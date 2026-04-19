# Trello API Automation Framework

<p align="center">
  <img src="https://img.shields.io/badge/Postman-FF6C37?style=flat-square&logo=postman&logoColor=white" />
  <img src="https://img.shields.io/badge/Newman-CLI-6BA539?style=flat-square&logo=npm&logoColor=white" />
  <img src="https://img.shields.io/badge/Chai-Assertions-A30701?style=flat-square" />
  <img src="https://img.shields.io/badge/JavaScript-ES6-F7DF1E?style=flat-square&logo=javascript&logoColor=black" />
  <img src="https://img.shields.io/badge/Trello-REST_API_v1-0052CC?style=flat-square&logo=trello&logoColor=white" />
</p>

End-to-end API automation suite for the **Trello REST API** — 8 requests, 4 entity domains, **168 Chai assertions**, and a hand-written recursive JSON Schema validator that catches field-level contract violations other test suites miss entirely.

---

## Key Points

- **Custom recursive JSON Schema validator** written in vanilla JavaScript — validates required fields, data types, regex patterns, and enum constraints per response, with error messages that report the exact failing field path (e.g. `root.prefs.permissionLevel: 'unknown' not in [private, org, public]`)
- **168 numbered Chai assertions** across 8 requests — sequential numbering makes it trivial to pinpoint which assertion failed in a Newman run
- **Dynamic ID chaining** via collection variables — entity IDs are captured from each response and injected into downstream requests automatically, no manual updates required between runs
- **Timestamped pre-request naming** — every entity gets a unique `Test_Board_1712345678` name on each run, preventing data collision across re-runs

---
## Project Stats
 
<table>
  <thead>
    <tr>
      <th align="left">Metric</th>
      <th align="left">Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Total requests</td>
      <td>8</td>
    </tr>
    <tr>
      <td>Total assertions</td>
      <td>168</td>
    </tr>
    <tr>
      <td>HTTP methods</td>
      <td>GET, POST, PUT</td>
    </tr>
    <tr>
      <td>Entities covered</td>
      <td>Board, List, Card, Checklist</td>
    </tr>
    <tr>
      <td>Schema validation</td>
      <td>Custom recursive JS validator</td>
    </tr>
    <tr>
      <td>ID format validation</td>
      <td><code>^[a-f0-9]{24}$</code></td>
    </tr>
    <tr>
      <td>Performance threshold</td>
      <td><strong>&lt; 300ms</strong> per request</td>
    </tr>
    <tr>
      <td>Report format</td>
      <td>Newman HTMLExtra</td>
    </tr>
  </tbody>
</table>
 
---
## Coverage

Every response is validated against:
- Field presence (required fields)
- Data types (`string`, `boolean`, `number`, `array`, `object`)
- Regex patterns — Trello's 24-char hex ID format: `^[a-f0-9]{24}$`
- Enum constraints — e.g. `permissionLevel` ∈ `{private, org, public}`
- Response time **< 300ms**

---

## Stack

| Tool | Purpose |
|---|---|
| Postman | Collection authoring, scripting |
| Newman + htmlextra | CLI execution, HTML reporting |
| Chai (`require('chai')`) | BDD assertions inside test scripts |
| JavaScript ES6 | Pre-request scripts, schema validator |
| Trello REST API v1 | System under test |

---

## Run It

**Prerequisites:** Node.js, a Trello API key + token ([get one here](https://trello.com/power-ups/admin))

```bash
# Install Newman
npm install -g newman newman-reporter-htmlextra

# Set your credentials in Trello_Env.json, then run:
newman run Trello_Automation.postman_collection.json \
  -e Trello_Env.json \
  -r htmlextra \
  --reporter-htmlextra-title "Trello API Automation Report" \
  --reporter-htmlextra-export ./report.html
```

Open `report.html` for a full breakdown of all 168 assertions with response bodies and timing.

---

## Schema Validator — Core Logic

```javascript
// Custom recursive validator — embedded in every test script
function validateSchema(data, schema, path = 'root') {
    const errors = [];

    (schema.required || []).forEach(field => {
        if (data[field] === undefined)
            errors.push(`Missing required field: ${path}.${field}`);
    });

    Object.keys(schema.properties || {}).forEach(field => {
        if (data[field] === undefined) return;
        const { type, pattern, enum: allowed, properties } = schema.properties[field];
        const actual = Array.isArray(data[field]) ? 'array' : typeof data[field];

        if (actual !== type)
            errors.push(`${path}.${field}: expected ${type}, got ${actual}`);
        if (pattern && !new RegExp(pattern).test(data[field]))
            errors.push(`${path}.${field}: failed pattern ${pattern}`);
        if (allowed && !allowed.includes(data[field]))
            errors.push(`${path}.${field}: not in [${allowed.join(', ')}]`);
        if (type === 'object' && properties)
            errors.push(...validateSchema(data[field], schema.properties[field], `${path}.${field}`));
    });

    return errors;
}

// Used in every pm.test block — failure message names the exact field
pm.test('[6] Schema valid', () => {
    const errors = validateSchema(json, entitySchema);
    expect(errors, errors.join(', ')).to.be.empty;
});
```

---

## Future Steps
 
1. **DELETE Operations & 404 Verification** — Add 3 requests (Delete Board, Delete List, Delete Card) with post-delete GET assertions verifying 404 responses; adds ~15 assertions
2. **Negative Testing Folder** — Create dedicated test suite for invalid tokens (401), missing fields (400), bad IDs (404), and duplicate operations; adds ~25 assertions
3. **GitHub Actions CI/CD** — Implement `.github/workflows/api-tests.yml` to auto-run Newman on push/PR with HTML artifact storage and Slack failure notifications
4. **Jenkins Pipeline Alternative** — Provide Dockerized `Jenkinsfile` with `Dockerfile` for enterprise deployments; includes Blue Ocean reports and email notifications
5. **Data-Driven Testing** — Enable CSV-based test runs via Newman `--iteration-data` flag; allows executing same test suite across multiple input datasets for bulk entity creation scenarios

---
