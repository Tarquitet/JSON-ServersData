# 📄 Sheets JSON Publisher

This project generates JSON files from Google Sheets using Google Apps Script and automatically publishes them to a GitHub repository.

The goal is to avoid querying Google Sheets directly from the frontend on every page load. Instead, the website consumes a lightweight static JSON file that is fast to read.

**Español:** [README.md](./README.md)

---

## ✨ Description

`Sheets JSON Publisher` turns editable Google Sheets data into versioned JSON files stored in GitHub.

The general workflow is:

```text
Google Sheets → Apps Script → JSON → GitHub → Frontend
```

This way, Google Sheets works as the editable data source, while GitHub acts as the publishing/delivery layer for the JSON files.

---

## 🧠 Why does this project exist?

Initially, the frontend could request Google Apps Script to generate the JSON on demand.

That approach had some issues:

- The end user had to wait for the JSON generation process.
- Google Sheets is not designed as a high-concurrency database.
- Responses could be slow.
- Google Apps Script quotas and execution limits could be consumed.
- Every visit could trigger an unnecessary read from the spreadsheet.

Because of that, the model was changed:

```text
End user
   ↓
Frontend
   ↓
Static JSON stored in GitHub
   ↑
Apps Script publishes the JSON
   ↑
Google Sheets as editable source
```

Now, users do not wait for data generation. They simply read an already generated file.

---

## 🏗 Architecture

```text
┌──────────────────┐
│  Google Sheets   │
└────────┬─────────┘
         │ data editing
         ▼
┌──────────────────┐
│  Apps Script     │
│  - reads data    │
│  - builds JSON   │
│  - commits file  │
└────────┬─────────┘
         │ GitHub API
         ▼
┌──────────────────┐
│  GitHub Repo     │
│  data/*.json     │
└────────┬─────────┘
         │ public read
         ▼
┌──────────────────┐
│  Frontend / Web  │
└──────────────────┘
```

---

## 📦 What does this project do?

- Reads data from one or more Google Sheets tabs.
- Transforms that data into JSON.
- Publishes the JSON to this repository using the GitHub Contents API.
- Allows updates:
  - manually through a Google Sheets button or menu,
  - scheduled every certain amount of time,
  - automatically after editing the sheet,
  - or by specific case/category.
- The frontend consumes the JSON from `raw.githubusercontent.com`.

---

## 📁 Repository structure

```text
/
├── data/
│   ├── cache.json
│   ├── case-a.json
│   ├── case-b.json
│   └── case-c.json
├── README.md
├── README.en.md
└── LICENSE
```

Generated JSON files are stored mainly inside:

```text
data/
```

---

## 🔗 Consuming the JSON

Currently, the frontend reads the data from GitHub Raw:

```text
https://raw.githubusercontent.com/YOUR_USER/YOUR_REPO/main/data/cache.json
```

JavaScript example:

```js
const JSON_URL =
  'https://raw.githubusercontent.com/YOUR_USER/YOUR_REPO/main/data/cache.json';

fetch(JSON_URL)
  .then(response => response.json())
  .then(data => {
    console.log(data);
  })
  .catch(error => {
    console.error('Error loading JSON:', error);
  });
```

Example using `async/await`:

```js
const JSON_URL =
  'https://raw.githubusercontent.com/YOUR_USER/YOUR_REPO/main/data/cache.json';

async function loadData() {
  try {
    const response = await fetch(JSON_URL);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const data = await response.json();

    console.log('Generated at:', data.generatedAt);
    console.log('Items:', data.count);

    return data.items;
  } catch (error) {
    console.error('Could not load JSON:', error);
    return [];
  }
}

loadData();
```

---

## 🧾 Expected JSON format

Each JSON file can include basic metadata and the main data entries.

Example:

```json
{
  "generatedAt": "2026-01-01T12:00:00.000Z",
  "source": "Google Sheets",
  "count": 2,
  "items": [
    {
      "id": 1,
      "name": "Example A",
      "status": "active"
    },
    {
      "id": 2,
      "name": "Example B",
      "status": "inactive"
    }
  ]
}
```

Main fields:

| Field | Description |
|---|---|
| `generatedAt` | Date and time when the JSON was generated. |
| `source` | Data source. |
| `count` | Number of generated items. |
| `items` | Array containing the main data entries. |

---

## ⚙️ Setup

### 1. GitHub repository

Create a repository where the JSON files will be published.

Example:

```text
https://github.com/YOUR_USER/YOUR_REPO
```

Inside the repository, the expected folder is:

```text
data/
```

---

### 2. Google Sheets

The spreadsheet is the data source.

Example structure:

| id | name | status |
|----|------|--------|
| 1  | Example A | active |
| 2  | Example B | inactive |

The first row is used as headers.

Each row becomes a JSON object.

Example:

```json
{
  "id": 1,
  "name": "Example A",
  "status": "active"
}
```

---

### 3. Google Apps Script

The script must be bound to the Google Sheet or have access to the corresponding spreadsheet.

Apps Script is responsible for:

1. Reading the sheet.
2. Converting the data into JSON.
3. Sending the file to GitHub using the GitHub Contents API.

---

### 4. GitHub token

A GitHub token with write access to the repository is required.

Recommendations:

- Use a fine-grained personal access token.
- Grant access only to the required repository.
- Grant read/write permission on contents.
- Do not expose the token in the frontend.
- Do not upload the token to the repository.

---

### 5. Script Properties

In Apps Script, store the token as a Script Property.

Path:

```text
Apps Script → Project Settings → Script Properties
```

Add a property:

| Name | Value |
|---|---|
| `GITHUB_TOKEN` | GitHub token |

Example:

```text
GITHUB_TOKEN = ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> ⚠️ Do not hardcode the token inside the script.

---

## 🔐 Security

- The GitHub token is stored in Apps Script as a Script Property.
- The token must not be exposed in the frontend.
- The token must not be committed to the repository.
- If the JSON contains sensitive information, it should not be published in a public repository.

For private data, consider:

- private repository with an authenticated backend,
- Cloudflare Access,
- Supabase,
- Firebase,
- custom API,
- signed URLs,
- or another access-control solution.

---

## 🕒 Automatic updates

Publishing can be executed using Google Apps Script triggers.

### Scheduled updates

A trigger can be created to publish the JSON every certain amount of time.

Examples:

```text
every 1 hour
every 3 hours
every 6 hours
every 12 hours
every 24 hours
```

---

### Update on edit

An installable `onEdit` trigger can also be used.

To avoid excessive publishing, debounce is recommended.

Conceptual flow:

```text
User edits sheet
   ↓
Apps Script detects change
   ↓
Waits 30-60 seconds
   ↓
If there are no more changes, publishes JSON
```

This avoids publishing on every keystroke.

---

## 🖱 Manual publishing

Publishing can also be executed manually from Google Sheets.

Options:

- custom menu,
- button inserted as image or drawing,
- separate functions per case.

Example menu:

```text
Publish JSON
├── Publish all
├── Publish case A
├── Publish case B
└── Publish case C
```

This allows generating specific files without updating everything.

---

## 📄 Case-based files

If there are multiple cases or categories, separate files can be generated:

```text
data/cache.json
data/case-a.json
data/case-b.json
data/case-c.json
```

Consumption example:

```js
fetch('https://raw.githubusercontent.com/YOUR_USER/YOUR_REPO/main/data/case-a.json')
  .then(response => response.json())
  .then(console.log);
```

---

## 🧪 Frontend usage example

```js
const DATA_URL =
  'https://raw.githubusercontent.com/YOUR_USER/YOUR_REPO/main/data/cache.json';

async function fetchData() {
  try {
    const response = await fetch(DATA_URL);

    if (!response.ok) {
      throw new Error(`HTTP error: ${response.status}`);
    }

    const data = await response.json();

    renderData(data.items);
  } catch (error) {
    console.error('Error fetching data:', error);
  }
}

function renderData(items) {
  const container = document.getElementById('app');

  if (!container) return;

  container.innerHTML = '';

  items.forEach(item => {
    const card = document.createElement('div');
    card.className = 'card';
    card.textContent = item.name || 'Unnamed';
    container.appendChild(card);
  });
}

fetchData();
```

---

## 🚀 Current status

Currently, the project consumes JSON files directly from:

```text
raw.githubusercontent.com
```

This is enough for now because the JSON files are lightweight and fast to read.

In the future, a CDN could be evaluated, such as:

- Cloudflare Workers,
- Cloudflare Pages,
- jsDelivr,
- Vercel Edge Functions,
- Netlify Edge Functions,
- or another caching platform.

This could improve:

- speed,
- cache control,
- invalidation,
- availability,
- HTTP headers,
- and strategies like `stale-while-revalidate`.

---

## 📌 Current limitations

- `raw.githubusercontent.com` may cache responses for a few minutes.
- Updates are not necessarily instant.
- Google Apps Script has execution limits and quotas.
- Google Sheets is not ideal as a high-concurrency database.
- If the data volume grows significantly, another solution should be considered.

---

## 🧰 Stack

- Google Sheets
- Google Apps Script
- GitHub API
- GitHub Raw
- JavaScript
- JSON

---

## 🧯 Troubleshooting

### JSON file does not appear

Make sure Apps Script successfully committed the file.

Check:

```text
data/cache.json
```

Also review the repository commit history.

---

### GitHub authentication error

Possible causes:

- invalid token,
- expired token,
- token without repository access,
- token without write permission,
- wrong username, repository, or branch.

Check the Script Property:

```text
GITHUB_TOKEN
```

---

### JSON is not updated

Check:

- whether the trigger is installed,
- whether the publish function is running,
- whether Apps Script execution logs show errors,
- whether the sheet contains valid data,
- whether GitHub is receiving the commit.

---

### Frontend shows old data

This may be caused by cache from:

- browser,
- GitHub Raw,
- service worker,
- CDN,
- local HTTP cache.

For testing, use DevTools with cache disabled.

In the future, if a CDN is used, cache invalidation or commit-based versioning can be implemented.

---

## 🚀 Roadmap

Possible future improvements:

- [ ] Use jsDelivr as a CDN.
- [ ] Use Cloudflare Workers for smart caching.
- [ ] Implement `stale-while-revalidate`.
- [ ] Publish a status endpoint.
- [ ] Generate commit-based versions for cache invalidation.
- [ ] Add publish logs.
- [ ] Add data validation before publishing.
- [ ] Add support for multiple sheets.
- [ ] Add support for multiple files.
- [ ] Migrate to Supabase or Firebase if real-time reading is required.
- [ ] Use GitHub Actions as an alternative pipeline.

---

## 📄 License

This project is licensed under the MIT License, unless otherwise stated.

---

## 🙌 Credits

This project uses Google Sheets as the editable data source, Google Apps Script as the JSON generator, and GitHub as the static file storage/delivery layer.
