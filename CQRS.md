## CQRS 패턴에 의한 데이터 통합

1. grab 서비스(8081)와 allocate 서비스(8082)를 각각 실행

```
cd grab
mvn spring-boot:run
```

```
cd allocate
mvn spring-boot:run
```

2. grab 요청

```sql
http localhost:8081/grabs taxiId=1 taxiNum="서울32저4703"
```

```sql
HTTP/1.1 201
Content-Type: application/json;charset=UTF-8
Date: Tue, 29 Mar 2022 04:12:23 GMT
Location: http://localhost:8081/grabs/1
Transfer-Encoding: chunked

{
    "_links": {
        "grab": {
            "href": "http://localhost:8081/grabs/1"
        },
        "self": {
            "href": "http://localhost:8081/grabs/1"
        }
    },
    "taxiId": 1,
    "taxiNum": "서울32저4703",
}
```

3. 카프카에서 이벤트를 확인

```
/usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic shopmall --from-beginning
```

```sql
{"eventType":"Grabbed","timestamp":"20220329041223","id":1,"taxiId":1,"taxiNum":"서울32저4703","me":true}
{"eventType":"Allocated","timestamp":"20220329041223","id":1,"grabId":1,"taxiId":1,"taxiNum":"서울32저4703","me":true}
```

4. grabView 서비스를 실행

```
cd grabView
mvn spring-boot:run

```

5. grabView의 Query Model을 통해 grab상태와 allocate상태를 통합 조회

- Query Model 은 발생한 모든 이벤트를 수신하여 자신만의 View로 데이터를 통합 조회 가능하게 함

```
http localhost:8090/grabStatuses
```

```sql
HTTP/1.1 200
Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 29 Mar 2022 04:13:00 GMT
Transfer-Encoding: chunked

{
    "_embedded": {
        "grabStatuses": [
            {
                "_links": {
                    "grabStatus": {
                        "href": "http://localhost:8090/grabStatuses/1"
                    },
                    "self": {
                        "href": "http://localhost:8090/grabStatuses/1"
                    }
                },
                "allocateId": 1,
                "allocateStatus": "Allocated",
                "grabStatus": "Grabbed",
                "taxiId": 1,
                "taxiNum": "서울32저4703",
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8090/profile/grabStatuses"
        },
        "search": {
            "href": "http://localhost:8090/grabStatuses/search"
        },
        "self": {
            "href": "http://localhost:8090/grabStatuses{?page,size,sort}",
            "templated": true
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}
```

    - grabView 에서 grab, allocate, taxi 상태를 통합 조회 가능함

6. Compensation Transaction 테스트(cancel grab)

- Taxi Grab 취소

```
http DELETE localhost:8081/grabs/1

```

```sql
HTTP/1.1 204
Date: Tue, 29 Mar 2022 04:13:27 GMT
```

- grab상태와 allocate상태 값을 확인

```
http localhost:8090/grabStatuses

```

```diff
HTTP/1.1 200
Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 29 Mar 2022 04:13:36 GMT
Transfer-Encoding: chunked

{
    "_embedded": {
        "grabStatuses": [
            {
                "_links": {
                    "grabStatus": {
                        "href": "http://localhost:8090/grabStatuses/1"
                    },
                    "self": {
                        "href": "http://localhost:8090/grabStatuses/1"
                    }
                },
                "allocateId": 1,
+                "allocateStatus": "AllocateCancelled",
+                "grabStatus": "GrabCancelled",
                "taxiId": 1,
                "taxiNum": "서울32저4703",
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8090/profile/grabStatuses"
        },
        "search": {
            "href": "http://localhost:8090/grabStatuses/search"
        },
        "self": {
            "href": "http://localhost:8090/grabStatuses{?page,size,sort}",
            "templated": true
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}
```

    - grab cancel 정보가 grabView에 전달되어 `grabStatus`, `allocateStatus` 모두 cancelled 로 상태 변경 된 것을 통합 조회 가능함
