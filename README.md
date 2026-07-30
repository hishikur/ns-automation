# ns-automation

OpenShift Namespace のライフサイクル管理を自動化する Ansible Playbook 集。  
AAP (Ansible Automation Platform) 2.7 の Job Template / Survey と連携して利用可能。

## Playbook 一覧

| Playbook | 概要 |
|---|---|
| `create-ns.yml` | Project 作成 + ResourceQuota + LimitRange + DB登録 |
| `delete-ns.yml` | Project 削除 + DB に削除日時を記録 |
| `list-entries.yml` | 管理テーブルの一覧表示 |
| `deploy-db.yml` | 管理用 PostgreSQL のデプロイ |

## 前提条件

- OpenShift クラスタへの認証（`KUBECONFIG` または AAP Credential）
- `kubernetes.core` Ansible コレクション
- `kubernetes` Python ライブラリ
- 管理用 PostgreSQL が `mgmt-db` namespace で稼働中（`create-ns` / `delete-ns` / `list-entries` で使用）

## 使い方

### Namespace 作成

```sh
ansible-playbook create-ns.yml -e namespace=my-app -e department=dev-team
```

Quota をカスタマイズする場合:

```sh
ansible-playbook create-ns.yml \
  -e namespace=my-app \
  -e department=dev-team \
  -e quota_cpu_limits=16 \
  -e quota_memory_limits=16Gi \
  -e quota_pods=50
```

### Namespace 削除

```sh
ansible-playbook delete-ns.yml -e namespace=my-app
```

### 管理テーブル一覧表示

```sh
# 稼働中のみ
ansible-playbook list-entries.yml

# 削除済みも含む
ansible-playbook list-entries.yml -e show_deleted=yes
```

### PostgreSQL デプロイ（初回セットアップ）

```sh
ansible-playbook deploy-db.yml -e namespace=mgmt-db
```

## デフォルト値

### ResourceQuota

| パラメータ | デフォルト |
|---|---|
| `quota_cpu_requests` | 4 |
| `quota_cpu_limits` | 8 |
| `quota_memory_requests` | 4Gi |
| `quota_memory_limits` | 8Gi |
| `quota_pods` | 20 |
| `quota_storage` | 10Gi |

### LimitRange（コンテナ既定値）

| パラメータ | デフォルト |
|---|---|
| `limitrange_default_cpu` | 500m |
| `limitrange_default_memory` | 512Mi |
| `limitrange_default_request_cpu` | 100m |
| `limitrange_default_request_memory` | 256Mi |

## ラベル

作成される Project には以下のラベルが付与される:

- `managed-by=fordemo` — 管理対象プロジェクトのフィルタ用
- `department=<部署コード>` — 部署別フィルタ用（英語表記）

## 管理データベース

`mgmt-db` namespace の PostgreSQL に `ns_list` テーブルで Namespace のライフサイクルを記録:

| カラム | 説明 |
|---|---|
| `department` | 管理部署 |
| `namespace_name` | Namespace 名 (UNIQUE) |
| `cpu_requests` / `cpu_limits` | CPU リソース |
| `memory_requests` / `memory_limits` | メモリリソース |
| `storage` | ストレージ |
| `created_at` | 作成日時 |
| `deleted_at` | 削除日時（NULL = 稼働中） |

## AAP 連携

AAP 2.7 の Job Template として登録済み:

| Job Template | Playbook | Survey |
|---|---|---|
| Create Namespace | `create-ns.yml` | 部署名、NS名、Quota 全項目 |
| Delete Namespace | `delete-ns.yml` | NS名 |
| List Namespaces | `list-entries.yml` | 削除済み表示オプション |

Execution Environment: `ee-supported-rhel9:latest`（`kubernetes.core` 同梱）
