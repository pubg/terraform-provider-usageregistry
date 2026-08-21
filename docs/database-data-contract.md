# Usage Registry 데이터 규약

이 문서는 `terraform-provider-usageregistry`가 DynamoDB에 저장하고 조회하는 데이터의 규약을 정의한다. Terraform resource뿐 아니라 DynamoDB에 직접 데이터를 쓰는 도구나 서비스도 이 규약을 따라야 한다.

이 문서에서 사용하는 용어의 의미는 다음과 같다.

- **필수**: provider가 정상적으로 읽고 갱신하려면 DynamoDB 항목에 항상 있어야 한다.
- **선택**: 생략할 수 있다. 값을 설정했다면 해당 필드의 검증 규약을 따라야 한다.
- **시스템 관리**: 호출자가 임의로 정하지 않고 provider가 생성하거나 갱신한다.
- **변경 불가**: Terraform에서 값을 바꾸면 기존 항목을 제자리에서 갱신하지 않고 삭제 후 다시 생성한다.

## 1. 테이블 규약

DynamoDB 테이블에는 String 타입의 파티션 키 `pk`와 정렬 키 `sk`가 있어야 한다. GSI나 LSI는 필요하지 않다. 모든 단건 읽기, Query, Scan과 BatchGet은 base table에서 strongly consistent read를 사용한다.

키와 내부 reference는 `#`을 구분자로 사용한다. 따라서 다음 값은 비어 있으면 안 되며 `#`을 포함할 수 없다.

- target type과 consumer type의 `name`
- usage record의 `target.type`, `target.id`, `consumer.type`, `consumer.id`
- 값이 설정된 `consumer.sub_id`

provider는 위 필드에 공백, 대소문자, URL 형식 또는 별도의 길이 제한을 적용하지 않는다. 이름과 ID 비교는 대소문자를 구분한다.

## 2. 엔터티와 키 구조

한 테이블에 type registry, 원본 usage record와 검색 index를 함께 저장한다.

| 엔터티 | `pk` | `sk` |
| --- | --- | --- |
| target type | `registry#target_type` | `<name>` |
| consumer type | `registry#consumer_type` | `<name>` |
| 원본 usage record | `target#<target.type>#<target.id>` | `consumer#<consumer.type>#<consumer.id>[#sub_id#<consumer.sub_id>]` |
| target 검색 index | `search_index#target#<target.type>#<target.id>` | `shard_main` |
| consumer 검색 index | `search_index#consumer#<consumer.type>#<consumer.id>` | `shard_main` |
| consumer sub-ID 검색 index | `search_index#consumer#<consumer.type>#<consumer.id>#sub_id#<consumer.sub_id>` | `shard_main` |

`consumer.sub_id`가 없는 record는 target과 consumer 검색 index에 포함된다. 값이 있는 record는 consumer sub-ID 검색 index에도 포함된다.

Terraform state와 import에서 사용하는 논리적 ID는 다음과 같다.

```text
record#<target.type>#<target.id>#<consumer.type>#<consumer.id>[#sub_id#<consumer.sub_id>]
```

Sub-ID가 없는 기존 ID 형식은 그대로 유지한다. 이 ID는 DynamoDB 속성으로 따로 저장하지 않는다.

## 3. Type registry의 공통 필드

Target type과 consumer type은 다음 시스템 관리 필드를 사용한다.

| 필드 | DB 필수 여부 | 타입 | 생성 규약 | 업데이트 규약 |
| --- | --- | --- | --- | --- |
| `version` | 필수 | Number | `1`로 시작한다. | 성공한 갱신마다 1씩 증가한다. |
| `created_at` | 필수 | String | 생성 시점의 UTC RFC3339 문자열을 저장한다. | 최초 값을 유지한다. |
| `updated_at` | 필수 | String | 생성 시 `created_at`과 같은 값을 저장한다. | 갱신 시점의 UTC RFC3339 문자열로 바꾼다. |
| `annotations` | 필수 | Map<String, String> | 입력을 그대로 저장하며, 생략하면 빈 맵을 저장한다. | 전체 맵을 새 값으로 교체한다. |

`annotations`의 키와 값은 provider가 해석하지 않지만 값의 타입은 모두 문자열이어야 한다. `metadata`라는 별도 필드는 사용하지 않는다.

## 4. Target type 규약

| 필드 | Terraform | DB | 타입 | 규약 |
| --- | --- | --- | --- | --- |
| `name` | 필수 | 필수 | String | 비어 있지 않아야 하며 `#`을 포함할 수 없다. `sk`와 같은 값이다. |
| `actions` | 필수 | 필수 | List<String> | 하나 이상이어야 하며 빈 문자열을 포함할 수 없다. 중복을 제거하고 사전순으로 저장한다. |
| `id_regex` | 선택 | 선택 | String | 유효한 Go 정규식이어야 한다. 생략하거나 빈 문자열이면 속성을 저장하지 않는다. |
| `id_regex_error_message` | 선택 | 선택 | String | ID 불일치 시 사용할 메시지이다. 생략하거나 빈 문자열이면 속성을 저장하지 않고 기본 메시지를 사용한다. |
| `annotations` | 선택 | 필수 | Map<String, String> | 생략하면 빈 맵을 저장한다. |
| `version`, `created_at`, `updated_at` | 읽기 전용 | 필수 | Number, String | 시스템이 관리한다. |

`id_regex`는 Go `regexp` 문법과 부분 일치 의미를 사용한다. 전체 ID를 제한하려면 `^`와 `$`를 명시해야 한다. `id_regex_error_message`만 설정하고 정규식을 생략하면 메시지는 아무 효과가 없다.

```json
{
  "pk": "registry#target_type",
  "sk": "vault_secret",
  "name": "vault_secret",
  "actions": ["read", "write"],
  "id_regex": "^kv/.+$",
  "annotations": { "owner": "platform" },
  "version": 1,
  "created_at": "2026-08-20T00:00:00Z",
  "updated_at": "2026-08-20T00:00:00Z"
}
```

## 5. Consumer type 규약

| 필드 | Terraform | DB | 타입 | 규약 |
| --- | --- | --- | --- | --- |
| `name` | 필수 | 필수 | String | 비어 있지 않아야 하며 `#`을 포함할 수 없다. `sk`와 같은 값이다. |
| `id_regex` | 선택 | 선택 | String | 유효한 Go 정규식이어야 한다. 생략하거나 빈 문자열이면 속성을 저장하지 않는다. |
| `id_regex_error_message` | 선택 | 선택 | String | ID 불일치 시 사용할 메시지이다. 생략하거나 빈 문자열이면 속성을 저장하지 않고 기본 메시지를 사용한다. |
| `annotations` | 선택 | 필수 | Map<String, String> | 생략하면 빈 맵을 저장한다. |
| `version`, `created_at`, `updated_at` | 읽기 전용 | 필수 | Number, String | 시스템이 관리한다. |

Consumer type의 `id_regex`는 `consumer.id`에만 적용하며 `consumer.sub_id`에는 적용하지 않는다.

## 6. 원본 Usage record 규약

| 필드 | Terraform | DB | 타입 | 규약 |
| --- | --- | --- | --- | --- |
| `target.type` | 필수 | 필수 | String | 등록된 target type이어야 한다. |
| `target.id` | 필수 | 필수 | String | Target type의 `id_regex`가 있으면 일치해야 한다. |
| `target.action` | 필수 | 필수 | String | Target type의 `actions`에 포함되어야 한다. |
| `target.version` | 선택 | 필수 | String | 생략하거나 빈 문자열이면 `latest`를 저장한다. |
| `consumer.type` | 필수 | 필수 | String | 등록된 consumer type이어야 한다. |
| `consumer.id` | 필수 | 필수 | String | Consumer type의 `id_regex`가 있으면 일치해야 한다. |
| `consumer.sub_id` | 선택 | 선택 | String | 같은 consumer ID의 여러 사용 관계를 구분한다. 설정했다면 비어 있거나 `#`을 포함할 수 없다. |
| `annotations` | 선택 | 필수 | Map<String, String> | 생략하면 빈 맵을 저장한다. |
| `entry_type` | 입력 불가 | 필수 | String | 항상 `usage_record`이다. |
| `version` | 읽기 전용 | 필수 | Number | `1`로 시작하고 갱신마다 1씩 증가한다. |
| `created_at`, `updated_at` | 읽기 전용 | 필수 | String | UTC RFC3339 문자열이며 시스템이 관리한다. |

```json
{
  "pk": "target#vault_secret#kv/team/db",
  "sk": "consumer#repository#https://git.example.com/team/repo#sub_id#prod",
  "entry_type": "usage_record",
  "target": {
    "type": "vault_secret",
    "id": "kv/team/db",
    "action": "read",
    "version": "latest"
  },
  "consumer": {
    "type": "repository",
    "id": "https://git.example.com/team/repo",
    "sub_id": "prod"
  },
  "annotations": { "owner": "platform" },
  "version": 1,
  "created_at": "2026-08-20T00:00:00Z",
  "updated_at": "2026-08-20T00:00:00Z"
}
```

`target.type`, `target.id`, `consumer.type`, `consumer.id`와 `consumer.sub_id`가 record identity를 구성한다. 이 필드들은 변경할 수 없으며 Terraform에서 변경하면 기존 record를 삭제하고 새 record를 생성한다. `target.action`, `target.version`과 `annotations`만 제자리에서 갱신할 수 있다.

## 7. 검색 index와 compact reference 규약

검색 index는 payload를 복제하지 않고 원본 record를 가리키는 String Set만 저장한다.

```json
{
  "pk": "search_index#consumer#repository#https://git.example.com/team/repo",
  "sk": "shard_main",
  "entry_type": "search_index",
  "record_refs": [
    "v1#t_type#vault_secret#t_id#kv/team/db#c_type#repository#c_id#https://git.example.com/team/repo#c_sub#prod"
  ]
}
```

`record_refs`의 실제 DynamoDB 타입은 String Set이다. V1 reference 형식은 다음과 같다.

```text
v1#t_type#<target.type>#t_id#<target.id>#c_type#<consumer.type>#c_id#<consumer.id>[#c_sub#<consumer.sub_id>]
```

- Set 속성 이름이 의미를 제공하므로 각 원소에 `record_ref#` 접두어를 붙이지 않는다.
- `v1`은 parser version이다. 기존 v1의 의미를 바꾸지 않으며 미래 구조는 v2로 추가한다.
- Label은 용량과 가독성을 함께 고려하여 `t_type`, `t_id`, `c_type`, `c_id`, `c_sub`를 사용한다.
- Parser는 version, label 순서, 필수 값과 예상하지 않은 token을 엄격히 검증한다.
- Reference 하나에서 원본의 `pk`와 `sk`를 모두 복원하므로 별도의 `record_pks`와 `record_sks`는 저장하지 않는다.
- 하나의 shard에 여러 reference version이 공존할 수 있다. 지원하지 않는 version은 데이터 손상 오류로 처리한다.

목록 조회는 검색 `pk`의 모든 shard를 strongly consistent Query하고 reference를 중복 제거한 뒤 정렬한다. 원본은 100개 단위의 strongly consistent BatchGet으로 읽으며 `UnprocessedKeys`를 모두 처리할 때까지 재시도한다. Query와 BatchGet 사이에 삭제된 원본은 결과에서 제외한다. Reference와 원본 identity가 다르면 검색 index 손상으로 처리한다.

## 8. 생성, 업데이트와 삭제 규약

### 8.1 생성

- 원본 record와 검색 index 2개 또는 3개를 하나의 `TransactWriteItems` 요청으로 처리한다.
- 원본은 같은 `pk`와 `sk`가 없을 때만 생성한다.
- 검색 index는 `ADD record_refs :reference`로 Set에 reference를 원자적으로 추가한다.
- 원본은 `version = 1`로 시작하며 `created_at`과 `updated_at`에 같은 값을 저장한다.

### 8.2 업데이트

- 원본의 현재 `version`이 Terraform state의 expected version과 같을 때만 갱신한다.
- 성공하면 `version`을 1 증가시키고 `updated_at`을 바꾸며 `created_at`은 유지한다.
- Mutable payload 변경은 identity와 reference를 바꾸지 않으므로 검색 index를 갱신하지 않는다.
- 현재 type 규약을 다시 검증하므로 type 규약을 강화한 뒤 기존 record가 이를 위반하면 payload 갱신도 실패한다.

### 8.3 삭제

- 원본의 expected version을 확인하면서 모든 검색 index에서 `DELETE record_refs :reference`를 하나의 트랜잭션으로 수행한다.
- 마지막 reference가 제거되면 String Set 속성은 없어지고 `entry_type`만 가진 shard document는 유지한다.
- 현재 target type과 consumer type의 존재 여부는 다시 검증하지 않는다.
- Type 삭제는 이를 참조하는 usage record를 cascade delete하지 않는다.

## 9. Shard 규약과 크기 제한

현재 writer는 항상 `sk = shard_main`만 사용한다. Reader는 `shard_main`을 직접 Get하지 않고 검색 `pk`의 모든 항목을 Query한다. 따라서 미래에 `shard_<n>`을 추가해 reference를 이동하거나 hash로 분배해도 조회 API를 바꾸지 않는다.

현재 구현은 자동 shard 분할을 지원하지 않는다. DynamoDB item의 400KB 제한을 넘는 ADD는 트랜잭션 전체를 실패시킨다. Migration dry-run은 검색 key별 예상 `shard_main` 크기를 계산하고 제한을 넘는 항목이 있으면 apply를 차단한다.

## 10. 기존 데이터 일괄 전환

`go run ./cmd/migrate-search-index`는 기본적으로 dry-run만 수행한다.

```shell
go run ./cmd/migrate-search-index \
  --table resource-usage-registry \
  --region ap-northeast-2 \
  --profile platform-prod
```

실제 변경에는 `--apply`와 정확한 table 이름 확인값이 모두 필요하다.

```shell
go run ./cmd/migrate-search-index \
  --table resource-usage-registry \
  --region ap-northeast-2 \
  --profile platform-prod \
  --apply \
  --confirm-table resource-usage-registry
```

운영 전환은 다음 순서를 따른다.

1. 모든 writer를 중지하고 DynamoDB backup 또는 PITR 상태를 확인한다.
2. Dry-run 결과에서 손상된 항목, orphan reference와 400KB 초과 항목이 없는지 확인한다.
3. Apply를 실행하여 compact reference를 추가하고 기존 `usage_record_reverse` 항목을 삭제한다.
4. 모든 원본의 검색 reference가 존재하고 reverse 항목이 0인지 확인한다.
5. 새 provider로 전환한 뒤 writer를 다시 시작한다.

Migration의 String Set ADD와 reverse 삭제는 record별 트랜잭션으로 처리하며 재실행할 수 있다. 이 저장소의 구현·검증 작업에서는 실제 DB migration을 실행하지 않는다.

## 11. DynamoDB를 직접 쓰는 구현의 필수 규약

직접 writer도 다음 규약을 따라야 한다.

1. 원본과 필요한 검색 index Set을 같은 트랜잭션으로 생성하거나 삭제한다.
2. Reference는 이 문서의 compact v1 encoder로 생성하며 문자열을 임의로 조합하지 않는다.
3. 검색 Set은 전체 값을 덮어쓰지 않고 DynamoDB `ADD`와 `DELETE`로 변경한다.
4. Identity가 아닌 payload를 갱신할 때는 원본 version 조건을 사용하고 검색 index를 변경하지 않는다.
5. 빈 `id_regex`와 `id_regex_error_message`는 속성 자체를 생략한다.
6. `target.version`을 생략하면 `latest`, `annotations`를 생략하면 빈 맵을 저장한다.

규약을 따르지 않은 직접 쓰기는 조회 누락, orphan reference, Terraform drift 또는 동시성 제어 우회를 일으킬 수 있다.
