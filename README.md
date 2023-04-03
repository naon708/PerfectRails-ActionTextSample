# Action Text

- WYSIWYGエディター
    - リッチテキストを編集するためのエディター

## ActionText関連のファイルを生成するコマンドを実行

```bash
% bin/rails action_text:install

# 変更
modified:   app/javascript/packs/application.js
modified:   package.json
modified:   yarn.lock

# 追加
app/assets/stylesheets/actiontext.scss
app/views/active_storage/
db/migrate/20230402111353_create_active_storage_tables.active_storage.rb
db/migrate/20230402111354_create_action_text_tables.action_text.rb
test/fixtures/action_text/
```

```ruby
# This migration comes from action_text (originally 20180528164100)
class CreateActionTextTables < ActiveRecord::Migration[6.0]
  def change
    create_table :action_text_rich_texts do |t|
      t.string     :name, null: false
      t.text       :body, size: :long
      t.references :record, null: false, polymorphic: true, index: false

      t.timestamps

      t.index [ :record_type, :record_id, :name ], name: "index_action_text_rich_texts_uniqueness", unique: true
    end
  end
end
```

```ruby
"@rails/actiontext": "^6.0.3",
"trix": "^1.2.0",
```

- WYSIWYGのJavaScript部分はTrixというnpmライブラリに依存しているのでインストールされた

## HTML

```html
<trix-toolbar>
<trix-editor>
```

## リッチテキストの投稿を作成

```bash
# ログ(整形済み)

Started POST "/messages"

Processing by MessagesController#create as HTML

Parameters: {"authenticity_token"=>"VtSPyp7QBhb94sZszz/bhQ2UqEe2Z7BdMd7U5Uz9ljgUXfhN8JaF4jy+8yDriQdLRG4NLIyt3fMhyOR8n2jK0A==", "message"=>{"content"=>"<div>### abc</div><div><strong><em><del>aaa</del></em></strong><strong>aaa</strong></div><h1>kkk</h1><div>lkkkk<br><br></div><blockquote>kkkk</blockquote><div><br></div><pre>code</pre><div><br><a href=\"https://www.google.com\">Google</a><br><br></div><ul><li>a<ul><li>b</li></ul></li><li>c</li></ul><ol><li>aa<ol><li>aaa</li><li>bb<ol><li>aaaa</li></ol></li></ol></li></ol><div><br></div>"}, "commit"=>"Create Message"}

Message Create  INSERT INTO "messages" ("created_at", "updated_at") VALUES (?, ?)  [["created_at", "2023-04-02 13:50:39.888083"], ["updated_at", "2023-04-02 13:50:39.888083"]]

ActionText::RichText Create  INSERT INTO "action_text_rich_texts" ("name", "body", "record_type", "record_id", "created_at", "updated_at") VALUES (?, ?, ?, ?, ?, ?)  [["name", "content"], ["body", "<div>### abc</div><div><strong><em><del>aaa</del></em></strong><strong>aaa</strong></div><h1>kkk</h1><div>lkkkk<br><br></div><blockquote>kkkk</blockquote><div><br></div><pre>code</pre><div><br><a href=\"https://www.google.com\">Google</a><br><br></div><ul><li>a<ul><li>b</li></ul></li><li>c</li></ul><ol><li>aa<ol><li>aaa</li><li>bb<ol><li>aaaa</li></ol></li></ol></li></ol><div><br></div>"], ["record_type", "Message"], ["record_id", 1], ["created_at", "2023-04-02 13:50:40.197187"], ["updated_at", "2023-04-02 13:50:40.197187"]]

ActiveStorage::Attachment Load  SELECT "active_storage_attachments".* FROM "active_storage_attachments" WHERE "active_storage_attachments"."record_id" = ? AND "active_storage_attachments"."record_type" = ? AND "active_storage_attachments"."name" = ?  [["record_id", 1], ["record_type", "ActionText::RichText"], ["name", "embeds"]]

Message Update  UPDATE "messages" SET "updated_at" = ? WHERE "messages"."id" = ?  [["updated_at", "2023-04-02 13:50:40.207441"], ["id", 1]]

Completed 302
```

- params→message→has_rich_textで指定した属性をキーにHTMLが格納されてる
- Messageレコードが何の変哲もなく作成される
- ActionTextのレコードがMessageテーブルのポリモーフィック関連付けでcreateされる
- 今度はActiveStorageがActionTextのポリモーフィックでLoad
- Messageのupdated_atを更新
- 302レスポンス

## リッチテキストの投稿詳細を取得

```bash
# ログ(整形後)
Started GET "/messages/1"

Processing by MessagesController#show as HTML

Message Load

ActionText::RichText Load
[["record_id", 1], ["record_type", "Message"], ["name", "content"], ["LIMIT", 1]]

Completed 200
```

## 一覧画面でリッチテキストを表示するようにするとN+1が発生する

```bash
Started GET "/messages"
Message Load

ActionText::RichText Load
ActionText::RichText Load
ActionText::RichText Load
ActionText::RichText Load
ActionText::RichText Load

Rendered messages/index.html.erb

Completed 200
```

```bash
# with_rich_text_#{name} メソッドでN+1を解消
Message Load (0.2ms)  SELECT "messages".* FROM "messages"

ActionText::RichText Load (3.0ms)  SELECT "action_text_rich_texts".* FROM "action_text_rich_texts" WHERE "action_text_rich_texts"."record_type" = ? AND "action_text_rich_texts"."name" = ? AND "action_text_rich_texts"."record_id" IN (?, ?, ?, ?, ?)  [["record_type", "Message"], ["name", "content"], ["record_id", 1], ["record_id", 2], ["record_id", 3], ["record_id", 4], ["record_id", 5]]
```

- preload

## カスタマイズするには

- CSSを編集
- `_blob.html.erb`を編集
- ツールバーをカスタマイズ(Trix)
    - [https://github.com/basecamp/trix](https://github.com/basecamp/trix)
