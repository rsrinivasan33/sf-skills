# Org Refresh to Local Skill

Sync Salesforce metadata from a live org to the local project when the SF CLI retrieve silently skips changed files due to source tracking issues.

---

## When to invoke

Trigger this skill when:
- The user says the org has changes but the local file looks stale after a retrieve
- `sf project retrieve start` succeeds but the local file is unchanged
- A retrieve fails with `UnsafeFilepathError` referencing a bad cached path
- The user wants to force-sync a specific metadata component from the org to disk

---

## Background: Why the SF CLI Retrieve Fails

The SF CLI tracks source state in `.sf/orgs/<orgId>/localSourceTracking/index` (a binary git-style index). When a retrieve is run with a bad `--output-dir` (e.g. a `/tmp/...` path), the CLI may embed that path in the index. On subsequent retrieves, the org rejects the request with:

```
UnsafeFilepathError: The filepath "../../../../../../../../tmp/.../SomeComponent.xml" contains unsafe character sequences
```

Even after deleting the bad output dir, the corrupt path persists in the binary index and blocks all retrieves for that org.

**Fix:** Delete the index file to force a rebuild:
```bash
rm .sf/orgs/<orgId>/localSourceTracking/index
```

The retrieve will then succeed, but the CLI may still silently skip files it considers up-to-date based on the rebuilt tracking state.

---

## Workflow

### Step 1 — Diagnose the retrieve issue

Check if the source tracking index is corrupt:
```bash
strings .sf/orgs/<orgId>/localSourceTracking/index | grep "tmp\|unsafe"
```

If a bad path is found, delete the index:
```bash
rm .sf/orgs/<orgId>/localSourceTracking/index
```

Retry the retrieve:
```bash
sf project retrieve start --metadata "<Type>:<ComponentName>" --target-org <alias>
```

### Step 2 — If CLI retrieve still skips the file

Use the SOAP Metadata API directly to read the live component:

```python
import urllib.request
import xml.etree.ElementTree as ET

instance_url = "https://<your-instance>.salesforce.com"
session_token = "<sf-cli-session-token>"  # from: sf org display --target-org <alias>

soap_body = """<?xml version="1.0" encoding="utf-8"?>
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:met="http://soap.sforce.com/2006/04/metadata">
  <soapenv:Header>
    <met:SessionHeader><met:sessionId>{token}</met:sessionId></met:SessionHeader>
  </soapenv:Header>
  <soapenv:Body>
    <met:readMetadata>
      <met:type>{metadata_type}</met:type>
      <met:fullNames>{component_name}</met:fullNames>
    </met:readMetadata>
  </soapenv:Body>
</soapenv:Envelope>""".format(token=session_token, metadata_type="GenAiPlugin", component_name="MyPlugin")

req = urllib.request.Request(
    f"{instance_url}/services/Soap/m/66.0",
    data=soap_body.encode(),
    headers={"Content-Type": "text/xml", "SOAPAction": "readMetadata"}
)

with urllib.request.urlopen(req) as resp:
    body = resp.read().decode()
```

**Note:** Use the SF CLI session token (from `sf org display`), NOT a Connected App OAuth token — the Metadata API SOAP endpoint requires a session token.

### Step 3 — Parse and write the local file

```python
ns = {'soap': 'http://schemas.xmlsoap.org/soap/envelope/', 'md': 'http://soap.sforce.com/2006/04/metadata'}
root = ET.fromstring(body)
records = root.find('.//md:records', ns)

def indent(elem, level=0):
    i = "\n" + level * "    "
    if len(elem):
        if not elem.text or not elem.text.strip():
            elem.text = i + "    "
        if not elem.tail or not elem.tail.strip():
            elem.tail = i
        for child in elem:
            indent(child, level + 1)
        if not child.tail or not child.tail.strip():
            child.tail = i
    else:
        if level and (not elem.tail or not elem.tail.strip()):
            elem.tail = i

indent(records)
content = ET.tostring(records, encoding='unicode')
content = content.replace(' xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="<MetadataType>"', '')
content = content.replace('<records', '<GenAiPlugin xmlns="http://soap.sforce.com/2006/04/metadata"')
content = content.replace('</records>', '</GenAiPlugin>')
output = '<?xml version="1.0" encoding="UTF-8"?>\n' + content

with open("<local-file-path>", "w") as f:
    f.write(output)
```

Adjust the root element name (`GenAiPlugin`, `GenAiPlanner`, `Bot`, etc.) to match the metadata type being retrieved.

### Step 4 — Verify the sync

Spot-check key fields in the written file match the org:
```bash
grep -n "<field-to-verify>" <local-file-path>
```

---

## Key Technical Notes

- The SOAP Metadata API endpoint is `{instanceUrl}/services/Soap/m/66.0`
- Get the session token via: `sf org display --target-org <alias>` → `Access Token` row
- The session token (starts with `00D...`) works for SOAP; Connected App OAuth tokens do NOT
- The `<records>` element in the SOAP response maps to the root metadata element in the local XML
- The `xsi:type` attribute on `<records>` must be stripped before writing the local file
- For Gov Cloud orgs, the instance URL is still the org's instance URL (e.g. `https://mysba--ocadev.sandbox.my.salesforce.com`) — the Gov Cloud API host (`api.gov.salesforce.com`) is only for the Einstein Agent API, not the Metadata API

---

## Supported Metadata Types (tested)

| Type | Root Element |
|---|---|
| `GenAiPlugin` | `<GenAiPlugin>` |
| `GenAiPlanner` | `<GenAiPlanner>` |
| `GenAiFunction` | `<GenAiFunction>` |
| `Bot` | `<Bot>` |
