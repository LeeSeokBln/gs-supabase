

`":company_id"`는 실제 회사 ID로 바꿔서 실행합니다.

## 0) 회사 ID 확인
조회 대상 회사의 `company_id` 확인

```sql
select id, name, status, created_at
from app.companies
order by id;
```

---

## 1) 월평균 대비 (사용이력)
UI `월평균 대비(%)` 검증

```sql
with current_storage as (
  select total_bytes
  from app.compute_storage_usage(:company_id)
),
monthly as (
  select
    date_trunc('month', day)::date as month,
    round(avg(storage_bytes) / 1024.0 / 1024.0 / 1024.0, 3) as avg_storage_gb
  from app.billing_usage_daily
  where company_id = :company_id
  group by 1
  order by 1 desc
),
past as (
  select avg_storage_gb
  from monthly
  offset 1
)
select
  round((select total_bytes from current_storage) / 1024.0 / 1024.0 / 1024.0, 3) as current_storage_gb,
  round(coalesce((select avg(avg_storage_gb) from past), 0), 3) as baseline_avg_gb,
  case
    when coalesce((select avg(avg_storage_gb) from past), 0) = 0 then null
    else round((((select total_bytes from current_storage) / 1024.0 / 1024.0 / 1024.0 - (select avg(avg_storage_gb) from past))
      / (select avg(avg_storage_gb) from past) * 100), 0)
  end as vs_monthly_avg_percent
;
```

- 계산식:
  - 월평균 대비(%) = ((현재 저장용량(GB) - 과거 월평균 저장용량(GB)) / 과거 월평균 저장용량(GB)) x 100

---

## 2) 연간 백업 활동 (사용이력 히트맵)
최근 1년 일자별 업로드 건수 검증

```sql
select
  date(minute_bucket at time zone 'Asia/Seoul') as day,
  sum(uploads_count) as upload_count
from app.backup_throughput
where company_id = :company_id
  and minute_bucket >= now() - interval '1 year'
group by 1
order by day desc;
```

- 계산식:
  - 일자별 활동값 = 해당 일자의 업로드 건수 합계
  - 연간 총 활동 = 최근 1년 일자별 업로드 건수 전체 합계

---

## 3) 일별 활동 추이 (사용이력, 최근 30일)
업로드/실패/삭제/다운로드/복구/설정변경 일별 건수 검증

```sql
with logs as (
  select
    date(ts at time zone 'Asia/Seoul') as day,
    case
      when message like 'UPLOAD_FAIL%' then 'upload_fail'
      when message like 'UPLOAD%' then 'upload'
      when message like 'DELETE%' then 'delete'
      when message like 'DOWNLOAD%' then 'download'
      when message like 'RESTORE%' then 'restore'
      when message like 'SETTINGS_CHANGE%' then 'settings'
      else 'other'
    end as kind
  from app.backup_logs
  where company_id = :company_id
    and ts >= now() - interval '30 days'
)
select
  day,
  sum(case when kind = 'upload' then 1 else 0 end) as upload,
  sum(case when kind = 'upload_fail' then 1 else 0 end) as upload_fail,
  sum(case when kind = 'delete' then 1 else 0 end) as delete_cnt,
  sum(case when kind = 'download' then 1 else 0 end) as download,
  sum(case when kind = 'restore' then 1 else 0 end) as restore,
  sum(case when kind = 'settings' then 1 else 0 end) as settings
from logs
group by day
order by day desc;
```

- 계산식:
  - 일별 활동 추이 = 일자별 (업로드 + 업로드실패 + 삭제 + 다운로드 + 복구 + 설정변경) 건수

---

## 4) 월별 저장용량 추이 (사용이력)
월별 평균 저장용량(GB) 검증

```sql
select
  date_trunc('month', day)::date as month,
  round(avg(storage_bytes) / 1024.0 / 1024.0 / 1024.0, 2) as avg_storage_gb
from app.billing_usage_daily
where company_id = :company_id
group by 1
order by month desc;
```

- 계산식:
  - 월별 저장용량 추이(GB) = 해당 월 일별 저장량(바이트) 평균 / 1,073,741,824

---

## 5) 호스트별 사용량 (사용이력)
호스트별 총 사용량(GB) 검증

```sql
with per_key as (
  select
    host,
    s3_key,
    max(coalesce(size_mb, 0)) as size_mb
  from app.file_events
  where company_id = :company_id
  group by host, s3_key
),
alive as (
  select p.*
  from per_key p
  where not exists (
    select 1
    from app.deleted_objects d
    where d.company_id = :company_id
      and d.host = p.host
      and d.key = p.s3_key
  )
)
select
  host,
  round((sum(size_mb) * 1024.0 * 1024.0 / 1024.0 / 1024.0 / 1024.0), 2) as usage_gb
from alive
group by host
order by usage_gb desc;
```

- 계산식:
  - 호스트별 사용량(GB) = (중복 제거 후 파일 크기(MB) 합계 x 1,048,576) / 1,073,741,824

---

## 6) 성공률 (대시보드/백업상태)
UI 성공률 표시 기준 검증

```sql
select count(*) as upload_count
from app.file_events
where company_id = :company_id;
```

- 계산식 (현재 UI 구현):
  - 성공률(%) = 업로드 건수 > 0 이면 100, 아니면 0

---

## 7) 최근 백업속도 (대시보드/백업상태)
최근 백업속도 계산 원본 검증

```sql
select
  host,
  uploaded_at,
  size_mb
from app.file_events
where company_id = :company_id
order by uploaded_at desc
limit 800;
```

- 계산식:
  - 각 업로드 순간속도(MB/s) = 현재 파일크기(MB) / (현재 업로드시각 - 이전 업로드시각(초))
  - 최근 백업속도(MB/s) = 최근 30개 순간속도의 평균

---

## 8) CPU / 메모리 / 네트워크 트래픽 (서버상태)
CPU/메모리/네트워크 지표 검증

```sql
select
  host,
  timestamp,
  cpu_percent,
  mem_percent,
  net_in_mbps,
  net_out_mbps
from app.host_metrics
where company_id = :company_id
  and timestamp >= now() - interval '7 days'
order by timestamp desc;
```

- 계산식:
  - CPU 사용률(%) = 수집값 그대로 사용
  - 메모리 사용률(%) = 수집값 그대로 사용
  - 네트워크 유입(Mbps) = 수집값 그대로 사용
  - 네트워크 유출(Mbps) = 수집값 그대로 사용
