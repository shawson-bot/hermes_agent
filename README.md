# Hermes Agent Profile Bundles

本 repository 用來保存與分享 Hermes 多代理工作流程的 profile 匯出檔，方便團隊成員在自己的 Hermes 環境中安裝、測試與部署。

## 1. 快速部署總覽

此 repository 建議放置兩組 profile bundles：

```text
hermes-profiles/
  README.md
  api-review-studio-profiles/
    api-review-orchestrator.tar.gz
    api-review-intake.tar.gz
    api-review-composer.tar.gz
    api-review-reviewer.tar.gz
  competition-proposal-profiles/
    competition-proposal-orchestrator.tar.gz
    competition-proposal-intake.tar.gz
    competition-proposal-source-reader.tar.gz
    competition-proposal-writer.tar.gz
    competition-proposal-reviewer.tar.gz
```

> 注意：`.tar.gz` 解開後，裡面應該直接是一個 Hermes profile folder，例如 `api-review-orchestrator/` 或 `competition-proposal-orchestrator/`。

## 2. 安裝 Profiles 到本機 Hermes

如果已經安裝 Hermes，通常會有：

```sh
~/.hermes/profiles
```

可先確認：

```sh
ls ~/.hermes/profiles
```

如果資料夾不存在，再建立：

```sh
mkdir -p ~/.hermes/profiles
```

`mkdir -p` 不會覆蓋既有資料夾，所以保留這行也是安全的。

### 安裝 API Review profiles

進入下載後的 API Review profile bundle folder：

```sh
cd /path/to/api-review-studio-profiles
```

解壓：

```sh
tar xzf api-review-orchestrator.tar.gz -C ~/.hermes/profiles
tar xzf api-review-intake.tar.gz -C ~/.hermes/profiles
tar xzf api-review-composer.tar.gz -C ~/.hermes/profiles
tar xzf api-review-reviewer.tar.gz -C ~/.hermes/profiles
```

確認：

```sh
ls ~/.hermes/profiles | grep api-review
```

### 安裝 Competition Proposal profiles

進入下載後的 Competition Proposal profile bundle folder：

```sh
cd /path/to/competition-proposal-profiles
```

解壓：

```sh
tar xzf competition-proposal-orchestrator.tar.gz -C ~/.hermes/profiles
tar xzf competition-proposal-intake.tar.gz -C ~/.hermes/profiles
tar xzf competition-proposal-source-reader.tar.gz -C ~/.hermes/profiles
tar xzf competition-proposal-writer.tar.gz -C ~/.hermes/profiles
tar xzf competition-proposal-reviewer.tar.gz -C ~/.hermes/profiles
```

確認：

```sh
ls ~/.hermes/profiles | grep competition-proposal
```

## 3. Provider / Credentials 設定

這些 profile 匯出檔不包含任何個人密鑰或本機狀態。每位使用者都需要在自己的裝置上重新設定：

- LLM provider
- API key 或 Vertex credentials
- Docker runtime
- 本機 project folder

可以用 Hermes UI / setup wizard 設定，也可以用 CLI。

## 4. Project Folder 設定

### API Review project folder

```sh
mkdir -p ~/hermes-projects/api_review/input_docs
```

每次 workflow 會建立獨立 run folder：

```text
~/hermes-projects/api_review/runs/<run_id>/
  input_docs/
  jobs/
  outputs/
  scratch/
  template/
```

### Competition Proposal project folder

```sh
mkdir -p ~/hermes-projects/competition_proposal/input_docs
```

每次 workflow 會建立獨立 run folder：

```text
~/hermes-projects/competition_proposal/runs/<run_id>/
  input_docs/
  jobs/
  outputs/
  scratch/
```

## 5. Docker / RTK Runtime 設定

兩組 workflow 都使用 RTK worker Docker image：

```text
hermes-report-sandbox:rtk-0.43.0
```

### API Review Docker image

API Review orchestrator profile 內已包含 RTK Docker build script：

```text
api-review-orchestrator/scripts/docker/rtk-report/build.sh
api-review-orchestrator/scripts/docker/rtk-report/Dockerfile
```

這個 image 是給 API Review 的 worker agents 使用，也就是：

```text
api-review-intake
api-review-composer
api-review-reviewer
```

用途是讓 sub-agent 在固定的 Docker runtime 內讀取 run folder、執行 RTK / Python / Node 相關工具，避免依賴每個使用者本機是否剛好有相同版本的工具。Orchestrator 本身建議跑在 local backend，負責建立每次 workflow 的 run folder，並把當次 run folder 掛載給 worker Docker 使用。


```sh
cd ~/.hermes/profiles/api-review-orchestrator
scripts/docker/rtk-report/build.sh
```

確認：

```sh
docker image ls | grep hermes-report-sandbox
docker run --rm hermes-report-sandbox:rtk-0.43.0 rtk --version
```

如果想使用不同 image name：

```sh
export HERMES_API_REVIEW_DOCKER_IMAGE=your-image-name:tag
```

### Competition Proposal Docker image

```sh
cd ~/.hermes/profiles/competition-proposal-orchestrator
scripts/docker/rtk-report/build.sh
```

確認：

```sh
docker run --rm hermes-report-sandbox:rtk-0.43.0 sh -lc 'rtk --version && python - <<PY
import fitz
import docx
from PIL import Image
print("document extraction deps ok")
PY'
```

Competition Proposal 的 Docker image 預設包含：

- RTK
- `pymupdf`
- `pymupdf4llm`
- `python-docx`
- `pillow`

因此預設可支援：

- Markdown / text
- DOCX
- 文字型 PDF

若需要掃描型 PDF 或圖片 OCR，可使用較重的 marker-pdf 版本：

```sh
cd ~/.hermes/profiles/competition-proposal-orchestrator
HERMES_COMPETITION_PROPOSAL_INSTALL_MARKER=1 scripts/docker/rtk-report/build.sh
```

> marker-pdf 可能會增加數 GB 的安裝與模型下載空間。若不安裝 marker-pdf，掃描型 PDF 或圖片 OCR 可能會被 Source Reader block 並要求補裝 OCR runtime 或提供文字版資料。

## 6. Portable Terminal Config 重設

如果 profile 是從其他裝置搬過來，建議先重設 terminal config，避免保留舊裝置的 Docker mount path。

### API Review

```sh
hermes -p api-review-orchestrator config set terminal.backend local
hermes -p api-review-orchestrator config set terminal.cwd '~/hermes-projects/api_review'
hermes -p api-review-orchestrator config set terminal.docker_volumes '[]'
hermes -p api-review-orchestrator config set terminal.docker_mount_cwd_to_workspace false

for p in api-review-intake api-review-composer api-review-reviewer; do
  hermes -p "$p" config set terminal.backend docker
  hermes -p "$p" config set terminal.cwd /workspace
  hermes -p "$p" config set terminal.docker_image hermes-report-sandbox:rtk-0.43.0
  hermes -p "$p" config set terminal.docker_volumes '[]'
  hermes -p "$p" config set terminal.docker_mount_cwd_to_workspace false
done
```

### Competition Proposal

```sh
hermes -p competition-proposal-orchestrator config set terminal.backend local
hermes -p competition-proposal-orchestrator config set terminal.cwd '~/hermes-projects/competition_proposal'
hermes -p competition-proposal-orchestrator config set terminal.docker_volumes '[]'
hermes -p competition-proposal-orchestrator config set terminal.docker_mount_cwd_to_workspace false

for p in competition-proposal-intake competition-proposal-source-reader competition-proposal-writer competition-proposal-reviewer; do
  hermes -p "$p" config set terminal.backend docker
  hermes -p "$p" config set terminal.cwd /workspace
  hermes -p "$p" config set terminal.docker_image hermes-report-sandbox:rtk-0.43.0
  hermes -p "$p" config set terminal.docker_volumes '[]'
  hermes -p "$p" config set terminal.docker_mount_cwd_to_workspace false
done
```

Orchestrator 會在每次 workflow start 時自動設定 run-specific Docker mounts。不要手動把某一次 run folder 寫死在 worker profile config 裡。

## 7. Optional：API Review GBrain / GStack

API Review orchestrator bundle 內含 optional setup script：

```sh
cd ~/.hermes/profiles/api-review-orchestrator
scripts/setup_gbrain_gstack.sh
```

如果需要 backend-local GBrain memory：

```sh
PATH="$HOME/.bun/bin:$PATH" gbrain init --pglite --no-embedding
```

確認：

```sh
PATH="$HOME/.bun/bin:$PATH" gbrain --version
find ~/.hermes/profiles/api-review-orchestrator/skills -maxdepth 1 -name 'gstack*' | wc -l
```

這是 optional。API Review workflow 不一定需要 GBrain/GStack 才能跑。

## 8. 功能說明

### API Review Workflow

用途：針對 API 文件進行 intake、撰寫審查報告、Reviewer 檢查，並由 Composer 根據 reviewer note 進行修訂。

包含 profiles：

```text
api-review-orchestrator
api-review-intake
api-review-composer
api-review-reviewer
```

流程：

```text
api_intake
  -> compose_api_review
    -> review_api_report
      -> revise_api_review
```

流程說明：

- `api-review-orchestrator`：使用者主要溝通入口，負責建立 workflow、管理任務狀態、回報進度與最終結果。
- `api-review-intake`：檢查使用者提供的 API 文件或來源資料是否可用，整理 intake 結果。
- `api-review-composer`：讀取 API 文件與 intake 結果，撰寫 API review report。
- `api-review-reviewer`：審查 composer 產出的報告，提出 reviewer note、風險提醒與修訂建議。
- `api-review-composer`：根據 reviewer note 進行修訂，輸出 final API review report。

建議輸入資料：

```text
api document
OpenAPI / Swagger spec
API markdown document
endpoint list
request / response examples
authentication notes
error code notes
integration notes
```

預期輸出：

```text
API review report
review notes
final revised API review report
```

### Competition Proposal Workflow

用途：針對競賽／計畫書資料進行 intake、source reading、mapping + outline、人工大綱審核、draft、review checklist、final draft 與 QA bank。

包含 profiles：

```text
competition-proposal-orchestrator
competition-proposal-intake
competition-proposal-source-reader
competition-proposal-writer
competition-proposal-reviewer
```

流程：

```text
Intake
  -> Source Reader
    -> Writer mapping.md + outline.md
      -> Human Outline Approval
        -> Writer draft.md
          -> Reviewer checklist.md
            -> Writer final draft + qa-bank
```

流程說明：

- `competition-proposal-orchestrator`：使用者主要溝通入口，負責建立 workflow、管理任務狀態、呈現 outline approval checkpoint，並回報最終成果。
- `competition-proposal-intake`：檢查可用 source files，標記缺少的建議 source roles，判斷是否需要 document extraction 或 OCR。
- `competition-proposal-source-reader`：讀取與萃取 source files，包含 Markdown/text、DOCX、文字型 PDF，以及可選的 OCR 圖片／掃描 PDF。
- `competition-proposal-writer`：產出 `mapping.md`、`outline.md`、`draft.md`，並在 reviewer checklist 後產出 final draft 與 QA bank。
- `competition-proposal-reviewer`：根據 guideline、source evidence、mapping、outline 與 draft 產出 `checklist.md`。

支援輸入格式：

```text
.md, .markdown, .txt
.pdf
.docx
.png, .jpg, .jpeg, .tif, .tiff, .webp
```

建議 source roles：

```text
guideline.*
company-data.*
product-data.*
```

workflow 可以少於三個檔案啟動，只要至少有一個可用 source。缺少的 role 會被標記為 `[待補充]`，不會直接阻擋流程。

## 9. 本機測試方式

### API Review 測試

建立測試資料夾：

```sh
mkdir -p ~/hermes-projects/api_review/input_docs
```

放入一個測試 API 文件：

```sh
cat > ~/hermes-projects/api_review/input_docs/test-api.md <<'EOF'
# Test API

## GET /users

Returns a list of users.

### Response

Response example:
{"users": []}
EOF
```

啟動 orchestrator：

```sh
hermes -p api-review-orchestrator
```

輸入：

```text
Start the API review workflow using the files in input_docs.
```

預期流程：

```text
Intake validates input
Composer writes API review report
Reviewer writes review notes
Composer revises final report
```

### Competition Proposal 測試

建立測試資料夾：

```sh
mkdir -p ~/hermes-projects/competition_proposal/input_docs
```

放入一個測試檔案：

```sh
cat > ~/hermes-projects/competition_proposal/input_docs/product-data.md <<'EOF'
# Product
This is a test product/source document.
EOF
```

啟動 orchestrator：

```sh
hermes -p competition-proposal-orchestrator
```

輸入：

```text
Start the competition proposal workflow for national-innovation-award using the files in input_docs.
```

當 workflow 停在大綱審核時，使用者可以回覆：

```text
我同意這個大綱，請繼續撰寫草稿。
```

如果不接受大綱，可以補充需求或上傳更多檔案，workflow 會重新進行 intake、source reader、mapping 與 outline。
