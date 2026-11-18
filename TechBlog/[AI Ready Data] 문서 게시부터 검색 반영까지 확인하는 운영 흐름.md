
    링크:
    https://blog.bespinglobal.com/post/ai-ready-data-operations-flow-from-document-publication-to-search-index-verification/?utm_source=rss&utm_medium=rss&utm_campaign=ai-ready-data-operations-flow-from-document-publication-to-search-index-verification

    ## AI 요약

    # AI Ready Data의 최신성 확보와 검색 동기화 요약

- **작성자**: 유수영 (Developer팀)
- **주제**: 문서 저장, 게시, 검색 인덱스 반영 단계의 분리 및 최신성 검증 방법

---

## ## 1. 핵심 개념: 저장과 검색 반영의 분리
새로운 데이터를 저장하는 것과 사용자가 최신 근거를 검색하는 것은 **별개의 단계**입니다. 문서 변환, 분할, 임베딩 생성이 완료되어 결과가 저장되더라도, **검색 인덱스 갱신**과 반영 검증이 완료되기 전까지는 사용자에게 이전 데이터가 노출될 수 있습니다. 따라서 **결과 저장**, **게시 완료**, **검색 반영 확인**의 단계를 명확히 구분해야 합니다.

---

## ## 2. 문서 개정 및 게시 관리
- **게시 식별자(Publication ID)**와 **동기화 식별자(Sync ID)**를 분리하여 관리해야 합니다.
- 단순히 가장 늦게 끝난 작업이 최신이 아니라, 현재 원문·분류·처리 정책(Processing Version)을 대조하여 게시 허용 여부를 판단해야 합니다.
- 새로운 개정 데이터를 반영할 때는 단순히 조각(Chunk)을 누적하는 것이 아니라, 이전 개정의 조각이 남지 않도록 **문서별 제공 범위를 교체(Replace)**해야 합니다.

```python
def replace_document(existing, document_id, replacement):
    if any(row["document_id"] != document_id for row in replacement):
        raise ValueError("다른 문서의 결과가 섞여 있습니다.")
    remaining = [dict(row) for row in existing if row["document_id"] != document_id]
    return remaining + [dict(row) for row in replacement]
```

---

## ## 3. 게시 스냅샷과 무결성 검증
동기화 과정에서 대상 목록이 변경되는 것을 막기 위해 **게시 스냅샷(Publication Snapshot)**을 고정하고, 해시 지문(SHA-256)을 통해 동일성을 검증합니다.

```python
import hashlib
import json

def freeze_publications(publications):
    keys = ("document_id", "publication_id", "event_id", "vector_rows", "type")
    snapshot = [{key: row[key] for key in keys} for row in publications]
    snapshot.sort(key=lambda row: row["document_id"])
    encoded = json.dumps(snapshot, ensure_ascii=False, sort_keys=True).encode("utf-8")
    return snapshot, hashlib.sha256(encoded).hexdigest()
```

- **빈 결과와 철회**: 새 개정에 벡터가 없거나 문서를 철회한 경우에도 이를 검색 인덱스에 반영(제거)해야 하며, 단순 오류로 인한 유실과 구분하여 처리해야 합니다.
- **갱신 상태 확인**: 단순히 갱신 요청(Sync Now)을 보낸 것에 만족하지 않고, 인덱스의 상세 상태(`ONLINE`, `ONLINE_NO_PENDING_UPDATE`)와 예상 행 수(`indexed_rows`)를 엄격히 대조합니다.

```python
def verify_sync(observation, expected_rows, frozen_hash, current_hash):
    if frozen_hash != current_hash:
        raise ValueError("동기화 중 게시 목록이 변경되었습니다.")
    if observation.get("state") not in {"ONLINE", "ONLINE_NO_PENDING_UPDATE"}:
        raise ValueError("상세 갱신 상태가 완료가 아닙니다.")
    if observation.get("indexed_rows") != expected_rows:
        raise ValueError("색인된 행 수가 예상과 다릅니다.")
    return {"publication_hash": frozen_hash, "indexed_rows": expected_rows}
```

---

## ## 4. 검색 품질 및 최종 대조
- **행과 열 검증**: 검색에 필요한 식별자, 개정, 본문, 출처 등의 열(Columns to index)과 유효한 벡터 값(`active`, `valid`)을 엄격히 검사합니다.
- **실제 질의 대조**: 최신 자료가 인덱스에 반영되었더라도 실제 대표 업무 질문을 통해 예상한 개정이 올바르게 반환되는지, 철회된 문서가 제외되었는지 최종 검토합니다.

---

## ## 5. 결론
최신성은 단순히 데이터를 저장하거나 파이프라인 실행 완료 표시를 보는 것으로 완성되지 않습니다. **저장·게시·반영**의 단계를 명확히 분리하고, 고정된 게시 목록을 바탕으로 실제 검색 결과를 검증할 때 비로소 신뢰할 수 있는 **AI Ready Data**가 구축됩니다.
    