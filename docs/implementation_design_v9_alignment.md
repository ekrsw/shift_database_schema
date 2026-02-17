# スキーマ定義書 v9 整合性対応 — 実装設計書

## 1. 概要

### 1.1 目的

`shift_database_schema_v9.md` で定義されたデータベース設計と、Prisma スキーマ・マイグレーション・API 実装の間に存在した相違点を解消し、データ整合性とシステム信頼性を確保する。

### 1.2 実施日

2026-02-17

### 1.3 対象スコープ

| 区分 | 内容 |
|------|------|
| DB マイグレーション | トリガー・部分ユニークインデックス・EXCLUDE 制約の追加 |
| API | 従業員更新・役割割当エンドポイントの修正 |
| バリデーション | 従業員更新スキーマの修正 |
| UI | 従業員基本情報編集フォームの修正 |

---

## 2. 対応した相違点一覧

### 2.1 重要度別サマリー

| # | 重要度 | 相違点 | 対応方針 | 状態 |
|---|--------|--------|----------|------|
| 1 | CRITICAL | シフト変更履歴トリガー未実装 | DB トリガー追加 | 対応済 |
| 2 | CRITICAL | `employee_function_roles` 部分ユニークインデックス未実装 | DB インデックス追加 + API トランザクション化 | 対応済 |
| 3 | CRITICAL | `employee_name_history` 部分ユニークインデックス未実装 | DB インデックス追加 | 対応済 |
| 4 | CRITICAL | `employee_name_history` EXCLUDE 制約未実装 | DB 制約追加 | 対応済 |
| 5 | IMPORTANT | PUT /api/employees/[id] で氏名変更時に履歴が作成されない | PUT から氏名変更を除外 | 対応済 |
| 6 | IMPORTANT | `role_type` 自動設定トリガー未実装 | アプリケーション層で代替（設計判断） | 対応見送り |
| 7 | IMPORTANT | `external_tools` / `employee_external_accounts` の API・UI 未実装 | 今回スコープ外 | 未対応 |
| 8 | MINOR | `employee_name_history.created_at` 精度差異 | 実装側（TIMESTAMP(3)）が高精度で問題なし | 対応不要 |
| 9 | MINOR | FK の ON DELETE 動作差異 | 実装側が適切 | 対応不要 |

### 2.2 設計判断

#### role_type 自動設定トリガーの見送り

スキーマ定義書では `trg_efr_set_role_type` トリガーによる自動設定を定義しているが、以下の理由でアプリケーション層での代替を採用した。

- **現状**: API コード（`POST /api/employees/[id]/roles`）が `function_roles.role_type` を取得して `employee_function_roles.role_type` に設定しており、機能的に同等
- **DB 直接操作への対応**: 部分ユニークインデックス `idx_efr_active_role_type` が `role_type` の不整合を検知するため、不正値の混入は防止される
- **Prisma との互換性**: Prisma は DB トリガーの副作用（カラム値の書き換え）を認識できず、ORM キャッシュとの不整合リスクがある

#### 氏名変更の PUT 除外

スキーマ定義書の `employee_name_history` は氏名変更の履歴管理を前提としている。PUT エンドポイントでの直接氏名変更は履歴が記録されず整合性が崩れるため、以下の方針とした。

- **氏名変更**: `POST /api/employees/[id]/name-history` エンドポイントに一本化
- **PUT /api/employees/[id]**: グループ・配属日・退職日のみ更新可能
- **UI**: 編集フォームで氏名は読み取り専用表示、「氏名変更履歴タブから行ってください」の案内を表示

#### 重複制御: 自動終了 + DB 制約

スキーマ定義書は部分ユニークインデックスで「重複をブロック」する設計だが、API の UX を考慮し以下のハイブリッド方式とした。

- **API 経由**: 同一 `role_type` の既存レコードを自動終了（`end_date` 設定）→ 新規作成（トランザクション内）
- **DB レベル**: 部分ユニークインデックスがフェイルセーフとして機能し、DB 直接操作での不整合を防止

---

## 3. DB マイグレーション設計

### 3.1 マイグレーションファイル

**パス**: `prisma/migrations/20260217000000_add_triggers_and_constraints/migration.sql`

### 3.2 シフト変更履歴トリガー

```
対象テーブル: shifts
トリガー名: trg_shift_change_history
関数名: record_shift_change()
発火タイミング: BEFORE UPDATE OR DELETE
粒度: FOR EACH ROW
```

#### 動作仕様

```
shifts テーブルへの UPDATE / DELETE
  │
  ├─ 1. shift_change_history から対象 shift_id の MAX(version) を取得
  ├─ 2. next_version = COALESCE(MAX(version), 0) + 1
  ├─ 3. OLD（変更前）の全カラム値を shift_change_history に INSERT
  │     - change_type = TG_OP（'UPDATE' or 'DELETE'）
  │     - version = next_version
  │     - changed_at = CURRENT_TIMESTAMP
  │
  ├─ DELETE の場合 → RETURN OLD
  └─ UPDATE の場合 → RETURN NEW
```

#### SQL

```sql
CREATE OR REPLACE FUNCTION record_shift_change()
RETURNS TRIGGER AS $$
DECLARE
  next_version integer;
BEGIN
  SELECT COALESCE(MAX(version), 0) + 1 INTO next_version
  FROM shift_change_history WHERE shift_id = OLD.id;

  INSERT INTO shift_change_history (
    shift_id, employee_id, shift_date, shift_code,
    start_time, end_time, is_holiday, is_paid_leave, is_remote,
    change_type, version, changed_at
  ) VALUES (
    OLD.id, OLD.employee_id, OLD.shift_date, OLD.shift_code,
    OLD.start_time, OLD.end_time, OLD.is_holiday, OLD.is_paid_leave, OLD.is_remote,
    TG_OP, next_version, CURRENT_TIMESTAMP
  );

  IF TG_OP = 'DELETE' THEN RETURN OLD; END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_shift_change_history
BEFORE UPDATE OR DELETE ON shifts
FOR EACH ROW EXECUTE FUNCTION record_shift_change();
```

#### 関連 API エンドポイント

以下のエンドポイントは shifts テーブルを UPDATE/DELETE するため、このトリガーによって自動的に履歴が記録される。

| エンドポイント | 操作 | 履歴記録 |
|---------------|------|----------|
| `PUT /api/shifts/[id]` | UPDATE | トリガーで自動記録 |
| `PUT /api/shifts/bulk` | UPDATE（複数） | 各行ごとにトリガー発火 |
| `POST /api/shifts/[id]/restore` | UPDATE（復元） | 復元操作自体もトリガーで記録 |

### 3.3 部分ユニークインデックス

#### idx_efr_active_role（同一役割の現行重複防止）

```sql
CREATE UNIQUE INDEX idx_efr_active_role
  ON employee_function_roles (employee_id, function_role_id)
  WHERE end_date IS NULL;
```

| 項目 | 値 |
|------|-----|
| テーブル | `employee_function_roles` |
| カラム | `(employee_id, function_role_id)` |
| 条件 | `end_date IS NULL`（現行レコードのみ） |
| 目的 | 同一従業員に同一役割の現行割当が2件以上存在することを防止 |

#### idx_efr_active_role_type（同一カテゴリの現行重複防止）

```sql
CREATE UNIQUE INDEX idx_efr_active_role_type
  ON employee_function_roles (employee_id, role_type)
  WHERE end_date IS NULL;
```

| 項目 | 値 |
|------|-----|
| テーブル | `employee_function_roles` |
| カラム | `(employee_id, role_type)` |
| 条件 | `end_date IS NULL`（現行レコードのみ） |
| 目的 | 同一従業員に同一カテゴリ（FUNCTION/AUTHORITY/POSITION）の現行割当が2件以上存在することを防止 |

**動作例**:

| 既存の現行役割 | 追加しようとする役割 | 結果 |
|---------------|--------------------|----|
| 受付 (FUNCTION) | 二次対応 (FUNCTION) | ブロック（API 経由では受付を自動終了→許可） |
| 受付 (FUNCTION) | SV (AUTHORITY) | 許可 |
| 課長 (POSITION) | 副部長 (POSITION) | ブロック（API 経由では課長を自動終了→許可） |

#### idx_enh_current（現行氏名は1件のみ）

```sql
CREATE UNIQUE INDEX idx_enh_current
  ON employee_name_history (employee_id)
  WHERE is_current = true;
```

| 項目 | 値 |
|------|-----|
| テーブル | `employee_name_history` |
| カラム | `(employee_id)` |
| 条件 | `is_current = true`（現行レコードのみ） |
| 目的 | 同一従業員の `is_current = true` レコードが2件以上存在することを防止 |

### 3.4 EXCLUDE 制約（有効期間重複防止）

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE employee_name_history
  ADD CONSTRAINT excl_enh_date_overlap
  EXCLUDE USING GiST (
    employee_id WITH =,
    daterange(valid_from, valid_to, '[]') WITH &&
  );
```

| 項目 | 値 |
|------|-----|
| テーブル | `employee_name_history` |
| 制約名 | `excl_enh_date_overlap` |
| インデックス種別 | GiST（`btree_gist` 拡張必須） |
| 目的 | 同一従業員の氏名履歴で有効期間が重複するレコードの作成を防止 |

**動作例**:

```
従業員ID=1 の氏名履歴:
  [2024-04-01, NULL)  山田太郎 (is_current=true)

→ valid_from=2024-01-01, valid_to=2024-05-01 を INSERT しようとすると
  期間 [2024-04-01, NULL) と [2024-01-01, 2024-05-01] が重複するためブロック
```

---

## 4. API 設計

### 4.1 PUT /api/employees/[id] — 従業員基本情報更新

**ファイル**: `src/app/api/employees/[id]/route.ts`

#### 変更前

```typescript
// リクエストボディ
{
  name: string,        // ← 氏名変更可能（履歴に記録されない）
  nameKana: string,    // ← カナ変更可能（履歴に記録されない）
  groupId: number,
  assignmentDate: string,
  terminationDate: string
}
```

#### 変更後

```typescript
// リクエストボディ
{
  groupId: number | null,       // 所属グループ
  assignmentDate: string | null, // 配属日
  terminationDate: string | null // 退職日
}
```

#### 変更理由

氏名変更は `employee_name_history` での履歴管理が必須であり、PUT での直接変更は履歴が記録されず整合性が崩れる。氏名変更は `POST /api/employees/[id]/name-history` に一本化。

### 4.2 POST /api/employees/[id]/roles — 役割割当

**ファイル**: `src/app/api/employees/[id]/roles/route.ts`

#### 変更前

```
1. functionRole を取得して role_type を確認
2. updateMany: 同一 role_type の既存レコードの end_date を設定
3. create: 新しい役割レコードを作成
   ※ 2 と 3 が別々のクエリ（トランザクション外）
```

#### 変更後

```
1. functionRole を取得して role_type を確認
2. $transaction 内で以下をアトミックに実行:
   a. updateMany: 同一 role_type の既存レコードの end_date を設定
   b. create: 新しい役割レコードを作成
3. ユニーク制約違反時は 409 Conflict を返却
```

#### シーケンス図

```
Client              API                    Prisma/$transaction        PostgreSQL
  │                  │                          │                        │
  │ POST /roles      │                          │                        │
  │ {functionRoleId} │                          │                        │
  │─────────────────>│                          │                        │
  │                  │ findUnique(functionRole)  │                        │
  │                  │─────────────────────────>│                        │
  │                  │ { roleType: "FUNCTION" }  │                        │
  │                  │<─────────────────────────│                        │
  │                  │                          │                        │
  │                  │ $transaction START        │                        │
  │                  │─────────────────────────>│ BEGIN                  │
  │                  │                          │───────────────────────>│
  │                  │                          │                        │
  │                  │ updateMany(endDate)       │ UPDATE ... end_date   │
  │                  │─────────────────────────>│───────────────────────>│
  │                  │                          │                        │
  │                  │ create(newRole)           │ INSERT ...             │
  │                  │─────────────────────────>│───────────────────────>│
  │                  │                          │        ┌───────────────│
  │                  │                          │        │ idx チェック   │
  │                  │                          │        └───────────────│
  │                  │                          │ COMMIT                 │
  │                  │                          │───────────────────────>│
  │                  │<─────────────────────────│                        │
  │ 201 Created      │                          │                        │
  │<─────────────────│                          │                        │
```

#### エラーハンドリング

| 状況 | レスポンス |
|------|-----------|
| 正常完了 | `201 Created` + 作成された役割レコード |
| 役割が見つからない | `404 Not Found` + `{ error: "役割が見つかりません" }` |
| ユニーク制約違反 | `409 Conflict` + `{ error: "この従業員には同じ役割カテゴリの現行レコードが既に存在します" }` |
| バリデーションエラー | `400 Bad Request` + Zod エラー詳細 |

### 4.3 氏名変更フロー（既存・変更なし）

氏名変更は既存の `POST /api/employees/[id]/name-history` エンドポイントで処理される（今回の修正対象外）。

```
Client → POST /api/employees/[id]/name-history
         { name, nameKana, validFrom, note }
           │
           ├─ 1. 既存の is_current=true レコードの valid_to, is_current を更新
           ├─ 2. 新しい履歴レコードを作成（is_current=true）
           └─ 3. employees テーブルの name, name_kana を更新
```

---

## 5. バリデーションスキーマ設計

**ファイル**: `src/lib/validations/employee.ts`

### 5.1 employeeUpdateSchema（変更あり）

```typescript
// 変更前
export const employeeUpdateSchema = z.object({
  name: z.string().min(1, "氏名は必須です").max(100),      // ← 削除
  nameKana: z.string().max(100).optional().nullable(),     // ← 削除
  groupId: z.number().optional().nullable(),
  assignmentDate: z.string().optional().nullable(),
  terminationDate: z.string().optional().nullable(),
});

// 変更後
export const employeeUpdateSchema = z.object({
  groupId: z.number().optional().nullable(),
  assignmentDate: z.string().optional().nullable(),
  terminationDate: z.string().optional().nullable(),
});
```

### 5.2 その他スキーマ（変更なし）

| スキーマ名 | 用途 | 変更 |
|-----------|------|------|
| `employeeCreateSchema` | 従業員新規作成 | なし（name/nameKana は初回登録時に必要） |
| `roleAssignSchema` | 役割割当 | なし |
| `roleEditSchema` | 役割編集 | なし |
| `nameChangeSchema` | 氏名変更 | なし |

---

## 6. UI コンポーネント設計

### 6.1 EmployeeBasicInfoForm（変更あり）

**ファイル**: `src/components/employees/employee-basic-info-form.tsx`

#### 表示モード（読み取り専用）— 変更なし

| フィールド | 表示 |
|-----------|------|
| 氏名 | テキスト表示 |
| フリガナ | テキスト表示（未設定時「—」） |
| グループ | テキスト表示 |
| 配属日 | テキスト表示 |
| 退職日 | テキスト表示 |

#### 編集モード — 変更あり

| フィールド | 変更前 | 変更後 |
|-----------|--------|--------|
| 氏名 | `<Input>` テキスト入力 | テキスト表示（読み取り専用） |
| フリガナ | `<Input>` テキスト入力 | テキスト表示（読み取り専用） |
| 案内文 | なし | 「※ 氏名の変更は「氏名変更履歴」タブから行ってください」 |
| グループ | `<Select>` ドロップダウン | 変更なし |
| 配属日 | `<Input type="date">` | 変更なし |
| 退職日 | `<Input type="date">` | 変更なし |

#### フォームの defaultValues

```typescript
// 変更前
defaultValues: {
  name: employee.name,
  nameKana: employee.nameKana,
  groupId: employee.groupId,
  assignmentDate: employee.assignmentDate,
  terminationDate: employee.terminationDate,
}

// 変更後
defaultValues: {
  groupId: employee.groupId,
  assignmentDate: employee.assignmentDate,
  terminationDate: employee.terminationDate,
}
```

---

## 7. DB オブジェクト一覧

本マイグレーションで追加された DB オブジェクトの一覧。

### 7.1 トリガー・関数

| 名前 | 種別 | 対象テーブル | 発火条件 |
|------|------|-------------|----------|
| `record_shift_change()` | 関数 | — | — |
| `trg_shift_change_history` | トリガー | `shifts` | BEFORE UPDATE OR DELETE |

### 7.2 インデックス

| 名前 | テーブル | カラム | 条件 | 種別 |
|------|---------|--------|------|------|
| `idx_efr_active_role` | `employee_function_roles` | `(employee_id, function_role_id)` | `end_date IS NULL` | UNIQUE |
| `idx_efr_active_role_type` | `employee_function_roles` | `(employee_id, role_type)` | `end_date IS NULL` | UNIQUE |
| `idx_enh_current` | `employee_name_history` | `(employee_id)` | `is_current = true` | UNIQUE |

### 7.3 制約

| 名前 | テーブル | 種別 | 内容 |
|------|---------|------|------|
| `excl_enh_date_overlap` | `employee_name_history` | EXCLUDE (GiST) | 同一 employee_id の有効期間重複禁止 |

### 7.4 拡張

| 名前 | 用途 |
|------|------|
| `btree_gist` | EXCLUDE 制約の GiST インデックスに必要 |

---

## 8. 修正ファイル一覧

| ファイル | 操作 | 修正内容 |
|---------|------|----------|
| `prisma/migrations/20260217000000_add_triggers_and_constraints/migration.sql` | 新規作成 | トリガー関数・トリガー・部分ユニークインデックス3件・EXCLUDE 制約・btree_gist 拡張 |
| `src/lib/validations/employee.ts` | 修正 | `employeeUpdateSchema` から `name` / `nameKana` を削除 |
| `src/app/api/employees/[id]/route.ts` | 修正 | PUT ハンドラの `data` から `name` / `nameKana` を削除 |
| `src/app/api/employees/[id]/roles/route.ts` | 修正 | `updateMany` + `create` を `$transaction` でラップ、ユニーク制約違反時の 409 エラーハンドリング追加 |
| `src/components/employees/employee-basic-info-form.tsx` | 修正 | 編集モードで氏名を読み取り専用表示に変更、案内文追加、フォーム defaultValues から name/nameKana を削除 |

---

## 9. 動作確認結果

### 9.1 テスト項目と結果

| # | 確認項目 | 方法 | 結果 |
|---|---------|------|------|
| 1 | シフト変更履歴トリガー（DB 直接） | SQL で直接 UPDATE を2回実行 | 変更前の値が version 1, 2 として自動記録された |
| 2 | シフト変更履歴トリガー（API 経由） | `PUT /api/shifts/1` で更新 | version 3 として自動記録された |
| 3 | PUT で氏名変更不可 | `PUT /api/employees/1` に name を含めて送信 | DB の氏名が変わらないことを確認 |
| 4 | 役割割当トランザクション | `POST /api/employees/1/roles` で同一 role_type を連続割当 | 既存が自動終了 → 新規が作成（アトミック） |
| 5 | 部分ユニークインデックス | DB で同一 role_type を直接 INSERT | `idx_efr_active_role_type` 違反でブロックされた |
| 6 | UI 編集フォーム | ブラウザで従業員詳細→編集ボタンクリック | 氏名が読み取り専用、案内文が表示された |
| 7 | シフト変更履歴画面 | ブラウザで `/shifts/history` を表示 | 3件の変更履歴が正しく表示された |

### 9.2 ビルド確認

```
$ npx next build
✓ ビルド成功（エラーなし）
```

### 9.3 マイグレーション確認

```
$ npx prisma migrate reset --force
Applying migration `20260216204756_init` ✓
Applying migration `20260217000000_add_triggers_and_constraints` ✓
Database reset successful
```

---

## 10. 残存課題

| # | 項目 | 重要度 | 内容 |
|---|------|--------|------|
| 1 | `external_tools` / `employee_external_accounts` の API・UI | IMPORTANT | テーブル定義はあるが API/UI が未実装。機能として利用不可 |
| 2 | `external_tools.tool_code` の UNIQUE 制約 | IMPORTANT | ツールコードの重複防止がない。将来的に追加が望ましい |
| 3 | `role_type` 自動設定トリガー | IMPORTANT | アプリ層で代替中。DB 直接操作時はインデックスで不整合を検知するが、role_type の値自体は正しくならない |

---

## 11. Prisma の制約事項

以下の DB オブジェクトは Prisma スキーマ（`schema.prisma`）では表現できず、マイグレーション SQL で直接管理している。

| DB オブジェクト | Prisma サポート | 管理方法 |
|---------------|----------------|----------|
| トリガー / 関数 | 未サポート | マイグレーション SQL |
| 部分ユニークインデックス (`WHERE` 句付き) | 未サポート | マイグレーション SQL |
| EXCLUDE 制約 | 未サポート | マイグレーション SQL |
| PostgreSQL 拡張 (`btree_gist`) | 未サポート | マイグレーション SQL |

`prisma migrate dev` で新しいマイグレーションを生成する際、これらの SQL 定義は自動生成されないため、手動で管理する必要がある。`prisma migrate diff` でスキーマの差分を確認する場合も、これらの DB オブジェクトは検出されない点に注意。
