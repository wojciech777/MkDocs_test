# Pagination, filtering, and sorting

All endpoints that return collections in Orion Platform 4.2 LTS use cursor pagination. The cursor model provides stable traversal of large sets even when new records are created during the iteration. This page describes the paging parameters, the filter syntax, the allowed sort orders, and the migration from the offset pagination of version 3.8.

## Paging parameters

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `limit` | integer | 25 | number of items per page, max. 100 |
| `starting_after` | string | — | cursor: return the items after the given one |
| `ending_before` | string | — | cursor: return the items before the given one |

The `starting_after` and `ending_before` parameters are mutually exclusive. Providing both fails with `ORN-1012`.

## Response structure

| Field | Type | Description |
| --- | --- | --- |
| `data` | array | the items of the page in the requested sort order |
| `has_more` | boolean | `true` when another page exists |
| `next_cursor` | string | cursor to pass in `starting_after`; `null` when `has_more` is `false` |
| `total_estimate` | integer | estimated number of items matching the filter |

!!! note "`total_estimate` is an estimate"
    The value comes from PostgreSQL planner statistics and may deviate from reality by a few percent. Do not use it for settlements or for computing a page count.

## Example of two consecutive requests

```bash title="First page"
curl "https://api.orion.example.com/v1/orders?limit=2&status=paid" \
  -H "X-Orion-Key: ok_live_9f2c41ab7d5e"
```

```json title="200 OK response, first page" hl_lines="16 17"
{
  "data": [
    {
      "id": "ord_01HQ8ZV3KXN4M2T9YB7C6D",
      "status": "paid",
      "created_at": "2026-04-12T09:31:04Z"
    },
    {
      "id": "ord_01HQ8ZV3KXN4M2T9YB7C5C",
      "status": "paid",
      "created_at": "2026-04-12T09:12:47Z"
    }
  ],
  "has_more": true,
  "next_cursor": "b3JkXzAxSFE4WlYzS1hOTTJUOVlCN0M1Qw",
  "total_estimate": 412
}
```

```bash title="Second page"
curl "https://api.orion.example.com/v1/orders?limit=2&status=paid&starting_after=b3JkXzAxSFE4WlYzS1hOTTJUOVlCN0M1Qw" \
  -H "X-Orion-Key: ok_live_9f2c41ab7d5e"
```

```json title="200 OK response, second page"
{
  "data": [
    {
      "id": "ord_01HQ8ZV3KXN4M2T9YB7C4B",
      "status": "paid",
      "created_at": "2026-04-12T08:58:03Z"
    }
  ],
  "has_more": false,
  "next_cursor": null,
  "total_estimate": 412
}
```

## Page traversal loop

=== "curl"

    ```bash
    cursor=""
    while :; do
      url="https://api.orion.example.com/v1/orders?limit=100&status=paid"
      [ -n "$cursor" ] && url="$url&starting_after=$cursor"
      body=$(curl -s "$url" -H "X-Orion-Key: ok_live_9f2c41ab7d5e")
      echo "$body" | jq -r '.data[].id'
      [ "$(echo "$body" | jq -r '.has_more')" = "true" ] || break
      cursor=$(echo "$body" | jq -r '.next_cursor')
    done
    ```

=== "Python"

    ```python
    cursor = None
    while True:
        page = client.orders.list(limit=100, status="paid", starting_after=cursor)
        for order in page.data:
            print(order.id)
        if not page.has_more:
            break
        cursor = page.next_cursor
    ```

=== "Java"

    ```java
    String cursor = null;
    while (true) {
        OrderPage page = client.orders().list(
            OrderListParams.builder()
                .limit(100)
                .status("paid")
                .startingAfter(cursor)
                .build());
        page.getData().forEach(o -> System.out.println(o.getId()));
        if (!page.getHasMore()) {
            break;
        }
        cursor = page.getNextCursor();
    }
    ```

## Cursor structure

A cursor is a base64url string containing the identifier of the last item on the page and the value of the sort field.

!!! danger "The cursor is opaque"
    Do not decode, modify, or generate cursors yourself. The format may change in any revision without notice. A hand crafted or expired cursor returns `ORN-1012`. Cursors remain valid for 24 hours.

## Sorting

The `sort` parameter takes the form `field:direction`, where the direction is `asc` or `desc`. The default is `created_at:desc`. Sorting by multiple fields is not supported.

| Resource | Allowed sort fields |
| --- | --- |
| `customers` | `created_at`, `updated_at`, `email` |
| `orders` | `created_at`, `amount_total`, `expires_at` |
| `payments` | `created_at`, `amount`, `captured_amount` |
| `events` | `created_at` |

An unsupported sort field returns `ORN-1004` with `error.field` set to `sort`.

## Filtering

Simple filters are written as `field=value`. Compound filters use operators in square brackets.

| Operator | Use | Example |
| --- | --- | --- |
| `[gte]` | value greater than or equal | `amount_total[gte]=100000` |
| `[lte]` | value less than or equal | `created_at[lte]=2026-04-01T00:00:00Z` |
| `[in]` | one of the values in a list, max. 20 | `status[in]=paid,fulfilled` |
| `[contains]` | a text fragment, min. 3 characters | `name[contains]=northwind` |

Filters are combined with a logical "and". Alternatives are only available through `[in]`. Filtering by `metadata` requires the full key path, for example `metadata[erp_id]=ORD/2026/04/118`.

## Limits and errors

| Code | HTTP | Cause |
| --- | --- | --- |
| `ORN-3004` | `400` | `limit` greater than 100 |
| `ORN-1012` | `400` | cursor invalid, expired, or inconsistent with `sort` |
| `ORN-1004` | `400` | unknown filter or sort field |
| `ORN-3001` | `429` | rate limit exceeded during the iteration |

!!! tip "Do not change `sort` mid-iteration"
    The cursor is tied to the sort order. Changing `sort` between pages invalidates the cursor and returns `ORN-1012`.

## Result stability

A cursor marks a position relative to the value of the sort field, not relative to a numeric offset. The practical consequences are:

- records created after the iteration started no longer shift the pages you have already visited,
- with `created_at:desc` new records will not appear in the current pass,
- a record modified during the iteration is returned in its new state, but still in its original position,
- a record that was deleted or anonymized disappears from the following pages.

For incremental processing, combine `sort=created_at:asc` with a `created_at[gte]` filter and remember the timestamp of the last processed record.

## Migration from 3.8 offset pagination

| 3.8 parameter | 4.2 equivalent | Notes |
| --- | --- | --- |
| `page` | none | replace with iteration over `next_cursor` |
| `per_page` | `limit` | maximum lowered from 500 to 100 |
| `offset` | `starting_after` | a cursor is required, not a number |
| `order_by` + `direction` | `sort=field:direction` | a single field instead of a list |
| `total_count` | `total_estimate` | an estimated value, not an exact one |

The offset endpoints were retired together with version 3.8 and are not available under `/v1`. Full procedure: [migration from 3.8 to 4.2](../../guides/migrations/from-3-8-to-4-2.md).

## See also

- [REST API basics](index.md)
- [Rate limits](rate-limits.md)
- [Error codes](error-codes.md)
- [Orders](resources/orders.md)
