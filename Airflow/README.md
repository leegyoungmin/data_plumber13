# Update Address and Send to Kafka DAG
이 DAG는 TDATA API에서 차량 위치 데이터를 가져와 Kakao API를 사용해 주소 정보를 업데이트하고, 처리된 데이터를 Kafka 토픽으로 전송합니다. 비동기 처리를 활용해 API 호출 성능을 최적화합니다.

## 필수 조건
Apache Airflow : 2.9.3

Kafka: 2.13

Python 3.8 이상 : 3.11.9

### 필요한 Python 패키지: requests, aiohttp, kafka-python

## Kafka 설정
Kafka 설정이 올바르게 구성되었는지 확인하기 위해 kafka 부트스트랩 서버와 토픽을 업데이트합니다.

```bash
KAFKA_BOOTSTRAP_SERVERS = 'server01:9092,server02:9092,server03:9092'
KAFKA_TOPIC = 'car_info'
```

## API 키 관리
API 키는 Airflow Variable로 관리합니다.

- tdata_apikey: TDATA API 키
- kakao_rest_api_key: Kakao API 키

Airflow CLI를 사용해 설정하는 법 : 

```bash
airflow variables -s tdata_apikey YOUR_TDATA_API_KEY
airflow variables -s kakao_rest_api_key YOUR_KAKAO_API_KEY
```

## DAG 정의
주소 데이터를 가져와 업데이트한 뒤 Kafka로 전송하는 두 가지 주요 작업으로 구성된 DAG를 15분 주기로 실행하도록 설정합니다.
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import requests
import json
import aiohttp
import asyncio
from kafka import KafkaProducer
from airflow.models import Variable

default_args = {
    'owner': 'DEProjTeam13',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=1),
}

dag = DAG(
    'update_address_and_send_to_kafka3',
    default_args=default_args,
    description='Fetch coordinates, update address, and send to Kafka',
    schedule_interval=timedelta(minutes=15),
    start_date=datetime(2024, 7, 23),
    catchup=False,
)

```

## 함수정의
### 비동기 주소 가져오기
Kakao API를 비동기적으로 호출해 주소를 가져옵니다.

```python
async def fetch_address(session, lat, lng, retries=3):
    url = f"https://dapi.kakao.com/v2/local/geo/coord2regioncode.json?x={lng}&y={lat}"
    headers = {"Authorization": f"KakaoAK {KAKAO_API_KEY}"}
    
    for attempt in range(retries):
        async with session.get(url, headers=headers) as response:
            if response.status == 200:
                return await response.json()
            else:
                print(f"시도 {attempt + 1} 실패: lat: {lat}, lng: {lng}, 상태 코드: {response.status}")
                await asyncio.sleep(0.5)
    return {"documents": []}
```

### 주소정보 업데이트
TDATA API에서 가져온 데이터의 주소 정보를 업데이트합니다.

```python
async def update_address(tdata_json):
    async with aiohttp.ClientSession() as session:
        tasks = []
        for index, i in enumerate(tdata_json):
            tasks.append(fetch_address(session, i['vhcleLat'], i['vhcleLot']))
            if len(tasks) == 100:
                results = await asyncio.gather(*tasks)
                for idx, result in enumerate(results):
                    if result and result.get('documents') and len(result['documents']) > 1:
                        doc = result['documents'][1]
                        if doc["region_1depth_name"] == "서울특별시":
                            addr = doc.get("region_2depth_name", "주소 없음")
                            tdata_json[index - 99 + idx].update({'addr': addr})
                        else:
                            tdata_json[index - 99 + idx] = None
                    else:
                        tdata_json[index - 99 + idx] = None
                tasks = []
        if tasks:
            results = await asyncio.gather(*tasks)
            for idx, result in enumerate(results):
                if result and result.get('documents') and len(result['documents']) > 1:
                    doc = result['documents'][1]
                    if doc["region_1depth_name"] == "서울특별시":
                        addr = doc.get("region_2depth_name", "주소 없음")
                        tdata_json[len(tdata_json) - len(results) + idx].update({'addr': addr})
                    else:
                        tdata_json[len(tdata_json) - len(results) + idx] = None
                else:
                    tdata_json[len(tdata_json) - len(results) + idx] = None
        tdata_json = [item for item in tdata_json if item is not None]
    return tdata_json

```

### 주소 가져오기 및 업데이트 

TDATA API에서 데이터를 가져와 Kakao API를 사용해 주소를 업데이트합니다.

```python
def fetch_and_update_address():
    response = requests.get(tdata_url)
    response.raise_for_status()
    tdata_json = response.json()
    updated_data = asyncio.run(update_address(tdata_json))
    return updated_data

```

### 데이터를 Kafka로 전송
업데이트된 데이터를 Kafka 토픽으로 전송합니다.

```python
def send_to_kafka(**kwargs):
    data = kwargs['ti'].xcom_pull(task_ids='fetch_and_update_address')
    if isinstance(data, list):
        producer = KafkaProducer(
            bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
        for item in data:
            producer.send(KAFKA_TOPIC, item)
        producer.flush()
    else:
        raise ValueError("fetch_and_update_address에서 리스트를 기대했지만 받은 타입: {}".format(type(data)))

```
### 작업 정의
PythonOperator를 사용해 작업을 정의합니다.
```python
fetch_and_update_task = PythonOperator(
    task_id='fetch_and_update_address',
    python_callable=fetch_and_update_address,
    dag=dag,
)

send_to_kafka_task = PythonOperator(
    task_id='send_to_kafka',
    python_callable=send_to_kafka,
    provide_context=True,
    dag=dag,
)

fetch_and_update_task >> send_to_kafka_task

```

## Airflow XCom

### XCom 활용 시 장점

1. 태스크 간 데이터 공유:
- 각 태스크를 독립적으로 실행하면서도 XCom을 통해 필요한 데이터를 공유하여 태스크 간의 협력과 데이터 의존성을 관리할 수 있습니다.
3. 유연한 데이터 전달:
- XCom을 사용하면 데이터 파이프라인에서 특정 태스크의 출력을 다른 태스크의 입력으로 쉽게 사용할 수 있어, 복잡한 데이터 흐름을 유연하게 구성할 수 있습니다.
4. 코드 재사용성 증가:
- XCom을 활용하면 공통 데이터를 재사용하는 여러 태스크를 보다 간단하게 구성할 수 있어, 코드의 중복을 줄이고 재사용성을 높일 수 있습니다.
4. 작업 상태 및 결과 관리:
- 태스크의 중간 결과나 상태를 저장하고 다른 태스크에서 이를 참조함으로써, 데이터 처리의 각 단계를 보다 명확하게 관리하고 모니터링할 수 있습니다.
  
### 중요 고려사항

1. 데이터 크기 제한:
- XCom은 메타데이터 데이터베이스에 저장되기 때문에 큰 데이터를 저장하기 적합하지 않습니다. 큰 데이터를 저장하기 위해서는 파일 경로 또는 데이터베이스 레코드 ID와 같은 참조를 이용해야 합니다.
2. 데이터 보안:
- XCom은 데이터를 데이터베이스에 저장하기 때문에 민감한 정보를 저장하면 보안이 위험할 수 있습니다.
3. XCom 키 관리:
- 기본 키를 사용하면 데이터 충돌이 일어날 수 있으므로 데이터를 저장할 때 명시적으로 키를 설정해 나중에 데이터를 쉽게 검색할 수 있도록 해야합니다.
4. 성능 문제:
- XCom을 과도하게 사용하면 메타데이터 데이터베이스 성능에 영향을 미칠 수 있습니다.
5. 의존성 관리:
- XCom은 DAG 내에서 작업 간 데이터 의존성을 관리하는 데 유용하지만, 복잡한 의존성을 만들지 않도록 주의해야합니다.

## Configuration as Code
### 이점
1. 버전 관리:
- Git과 같은 버전 관리 시스템을 통해 변경 사항을 추적할 수 있습니다.
2. 재현성:
-코드로 작성된 설정은 어디서나 동일하게 재현될 수 있습니다.
3. 자동화:
-설정을 CI/CD 파이프라인에 통합해 자동으로 배포될 수 있습니다.
4. 협업:
- 코드 리뷰를 통해 협업할 수 있습니다.
5. 문서화:
- 설정 파일 자체가 문서 역할을 합니다.

### Airflow CLI 명령어 : 변수를 설정하고 관리하는 명령어
```python
# 변수 설정
airflow variables -s <KEY> <VALUE>

# 변수 가져오기
airflow variables -g <KEY>

# 변수 삭제
airflow variables -x <KEY>

# 모든 변수 나열
airflow variables -l

```
### Dag 관련 다양한  명령어 
```python
#DAG를 실행
airflow dags trigger example_dag

#DAG를 중지(일시 정지)
airflow dags pause example_dag

#중지된 DAG를 재개
airflow dags unpause <dag_id>

#모든 DAG와 그 상태를 확인
airflow dags list

#특정 DAG의 실행 상태를 확인
airflow dags list-runs -d <dag_id>
```


## 동기와 비동기 API 호출 비교
### 동기 방식
라이브러리: requests

실행 시간: 약 50-60초

<img width="251" alt="Synchronous" src="https://github.com/user-attachments/assets/8e5426a3-f8fa-4303-b341-e1427793e647">

단점: 순차 처리로 인해 대량의 API 호출을 처리하기 비효율적입니다.

### 비동기 방식
라이브러리: aiohttp 및 asyncio

실행 시간: 약 5초

<img width="250" alt="Asynchronous" src="https://github.com/user-attachments/assets/15833b5d-b2c8-47c7-a2e9-b0705505c760">

장점: 병렬 처리를 통해 대량의 API 호출을 효율적으로 처리할 수 있습니다.

### 결론
대규모 API 호출을 처리할 때는 aiohttp와 asyncio를 활용한 비동기 처리가 동기 처리에 비해 훨씬 효율적입니다.비동기 처리시, API 서버에 응답 시간에 따라 delay를 주어야 하는 경우도 고려해야 하고, 짧은 시간동안 많은 요청을 할 시에 정상적인 응답을 못받는 경우가 있지만 위 사항에 대한 조치를 할 경우 비동기 처리를 통해 처리 시간을 줄이고 성능을 향상시킬 수 있습니다.
