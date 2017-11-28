
# 배포 방법 정리

## 릴리스 노트

> PR : #xx release-171205
> v1.2x.0.0

이번 배포에 포함된 내용은 아래와 같습니다.

-   할증 필터(`ExtraFeeFilter`) 및 할증 필터 팩토리 추가 (`ExtraFeeFilterFactory`)
    - POS API 배송 생성 / 예상 금액 조회 / 배송 도착지 정보 수정
    - 화주사 API 배송 생성 / 예상 금액 조회 / 배송 도착지 정보 수정
-   상점 모델에 위탁사 할증 적용 여부 필드(`vroong_extra_fee_applicable`) 추가
    - 모든 타입의 상점 생성, 수정, 조회 시 적용
-   배송 도착지 정보 수정 시 할증 업데이트 수정
    - 기존: 시간/지역 할증을 모두 업데이트
    - 변경: 지역 할증만을 업데이트
    - `ExtraFeeValueObject`에 `ExtraFeeType` 타입의 `$type` 필드 추가
    - `deliveries.extra_fees`에 직렬화된 JSON에 type 필드가 추가됨
    ```json
    // 기존
    [
        {
            "title": "기상할증", 
            "amount": 500, 
            "vroong_extra_charge_amount": null
        }
    ]

    // 변경
    [
        {
            "type": "WEATHER",  // or "REGIONS", "WEEKLYHOURS"
            "title": "기상할증", 
            "amount": 500, 
            "vroong_extra_charge_amount": null
        }
    ]
    ```

## 배포 가이드

### 1. 빌드

최신 Swagger Spec으로 vroong-api PHP Library를 빌드하고, `master` 브랜치에 배포해야 합니다.

```bash
~/vroong-lastmile-api $ swagger-codegen ...
```

- [ ] Done

새로 생성한 부릉 API 라이브러리를 추가하여 버전 잠금을 하고, 변경 내용을 배포 브랜치에 머지해야합니다.

```bash
~/prime-main-server $ composer require meshkorea/vroong-api:dev-master
~/prime-main-server $ git add . && git commit -m 'RIDER-262 부릉 라이브러리 교체' && git push
~/prime-main-server $ git pull && composer install
```

배포 브랜치를 만들고, `master` 브랜치에 머지한 후 빌드합니다.

```bash
(local)$ DEPLOY_VERSION="v1.2x.0.0"
(local)$ DEPLOY_BUILD_PATH="build-$DEPLOY_VERSION"
(local)$ git clone https://github.com/meshkorea/prime-main-server.git $DEPLOY_BUILD_PATH
(local)$ cd $DEPLOY_BUILD_PATH
(local)$ composer install
(local)$ eb init
```

### 2. 코드배포

`prime-accounting`에 먼저 배포합니다.

```bash
(local)$ eb deploy prime-accounting --label="$DEPLOY_VERSION" --verbose --timeout=30
```

### 3. 테이블 스키마 마이그레이션

`prime-accounting` 배포후, 해당 인스턴스에서 DB migration 진행합니다. 

```bash
(remote) $sudo -E -u webapp php /var/www/html/artisan migrate
# 2017_11_01_000100_add_vroong_monitoring_partner_extra_fee_activated_to_store
# 2017_11_22_000100_drop_mesh_account_number_on_stores_table
# 2017_11_22_000200_add_admin_memo_and_soft_deletes_on_releases_table
```

마이그레이션 진행 후 `stores` 테이블에 `vroong_extra_fee_applicable` 필드를 확인합니다.

#### 3.1. 연동 확인

부릉, 포인트 연동 확인 스크립트

#### 3.2. 커맨드 실행

....

커맨드 실행 결과를 검증하기 위한 방법 기술

### 4. 코드 배포 2

`prime-cron-worker-php7`, `prime-queue-worker-php7`, `prime-production-php7` 순으로 배포합니다.

```bash
(local) eb deploy prime-cron-worker-php7 --version="$DEPLOY_VERSION" --verbose --timeout=30
(local) eb deploy prime-queue-worker-php7 --version="$DEPLOY_VERSION" --verbose --timeout=30
(local) eb deploy prime-production-php7 --version="$DEPLOY_VERSION" --verbose --timeout=30
```

큐 워커 프로세스 재시작 여부를 확인합니다 (2대).

```bash
(queue worker) $ sudo -E -u webapp /usr/local/bin/supervisorctl status
```

### 5. 모니터링
