# 6. Salesforce Integration

---

## 6.1 Outbound Messages need a SOAP ACK

**Symptom:** one record change triggers the downstream action (e.g. a WhatsApp message) **three or more times**.

**Cause:** a plain HTTP 200 isn't enough. Salesforce parses the response body for `<Ack>true</Ack>`, and without it retries the notification on a backoff schedule for up to 24 hours.

**Fix:** Webhook with `responseMode: responseNode`. The **first** node after it is *Respond to Webhook* with `Content-Type: text/xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
  <soapenv:Body>
    <notificationsResponse xmlns="http://soap.sforce.com/2005/09/outbound">
      <Ack>true</Ack>
    </notificationsResponse>
  </soapenv:Body>
</soapenv:Envelope>
```

The rest of the flow runs after the ACK.

---

## 6.2 Outbound Message delivery is queued

Same flow, same record shape, 15 minutes apart: one delivery took **about a second**, the next **more than 4 minutes**. The queue is shared org-wide.

**Rules:**
- Wait at least 5 minutes before concluding a flow didn't fire.
- **Don't delete a dedup row** for a record whose message may still be queued. That row is the only thing between a late delivery and a duplicate customer message.

---

## 6.3 Phone matching with LIKE loses formatted numbers

**Symptom:** some calls get "no AI summary". No error, no skip row.

**Cause:** `Phone LIKE '%<9 digits>%'` can't match `+1 555-012-3456`, because the formatted number has no nine contiguous digits. That left **~1.5% of open opportunities** unreachable.

**Fix: narrow in SOQL, decide in code.**
- Query on the **last 4 digits, anchored to the end** (`LIKE '%3456'`). The unanchored form returned ~20 candidates, close to the cap. The anchored form returned a handful.
- Normalise candidates to digits in a Code node and match the last 9 exactly.
- **Name the Code node the same as the old node.** Downstream `$('Node Name')` references keep working without changes.

Result: ~96% of the unreachable records recovered, zero regressions on a control set.

---

## 6.4 Converted Leads can't be updated

`CANNOT_UPDATE_CONVERTED_LEAD` is a platform restriction. In one backfill, **more than half the targets** were converted. Add `AND IsConverted = false` to every Lead write job, and report updatable and frozen counts separately. Fix attribution on the converted record (Opportunity/Contact) instead.

---

## 6.5 Push, don't poll

For status-change automations, a record-triggered Flow plus Outbound Message, or a Platform Event, beats a scheduled SOQL poll. You avoid missed transitions between polls, wasted API calls and race windows.
