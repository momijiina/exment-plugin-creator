# Exmentプラグイン開発ガイド

## Exmentプラグインの基本構造

Exmentプラグインは、主に以下の要素で構成されます。

*   **`config.json`**: プラグインのメタデータ（名前、バージョン、UUID、タイプなど）を定義するファイルです。
*   **`Plugin.php`**: プラグインの主要なロジックを記述するPHPファイルです。プラグインのタイプに応じて、特定の基底クラスを継承します。
*   **ビューファイル**: プラグインがUIを持つ場合（例: ページ、ビュー、ダッシュボード）、対応するBladeテンプレートファイルなどです。
*   **その他のファイル**: CSS、JavaScript、画像などのアセットファイルや、追加のPHPクラスなどです。

## プラグインの種類と作成方法

Exmentには、様々な種類のプラグインがあり、それぞれ異なる目的と作成方法を持っています。以下に主要なプラグインタイプとその概要、作成方法を詳述します。

### トリガー

トリガープラグインは、Exment内の特定のイベント（例: データの作成、更新、削除）が発生した際に自動的に実行される処理を定義します。これにより、データの整合性維持、外部システム連携、通知の送信など、様々な自動化を実現できます。

#### 作成方法

1.  **`Plugin.php`ファイルの作成**: `app/Plugins` ディレクトリ内に、プラグイン名に対応するディレクトリを作成し、その中に`Plugin.php`ファイルを作成します。
2.  **`config.json`の設定**: `config.json`ファイルで`plugin_type`を`trigger`に設定します。
3.  **トリガーロジックの実装**: `Plugin.php`内で、`PluginTriggerBase`を継承したクラスを作成し、`handle`メソッドにトリガー処理のロジックを記述します。`handle`メソッドは、イベント発生時に自動的に呼び出されます。

#### 例: データ作成時にログを記録するトリガー

```php
<?php

namespace App\Plugins\LogTrigger;

use Exceedone\Exment\Services\Plugin\PluginTriggerBase;
use Exceedone\Exment\Model\CustomTable;
use Illuminate\Support\Facades\Log;

class Plugin extends PluginTriggerBase
{
    /**
     * トリガーが実行される際に呼び出される処理です。
     *
     * @param string $event イベント名 (例: create, update, delete)
     * @param array $data イベントに関連するデータ
     * @return void
     */
    public function handle($event, $data)
    {
        Log::info("Exment Trigger Event: " . $event, $data);

        // 例: 特定のテーブルにデータが作成されたら、別の処理を実行
        if ($event === 'create' && $data['table_name'] === 'your_table_name') {
            // ここにカスタムロジックを記述
            // 例: 関連するデータを更新する、外部APIを呼び出すなど
        }
    }
}
```

#### 注意事項

*   トリガーは、同期的に実行されるため、処理に時間がかかる場合はExmentのパフォーマンスに影響を与える可能性があります。非同期処理が必要な場合は、キューやバッチ処理の利用を検討してください。
*   `config.json`で`event_type`を指定することで、特定のイベントタイプ（例: `create`, `update`, `delete`）のみにトリガーを限定できます。

### ページ

ページプラグインは、Exment内に完全に独立した新しいページを追加します。これにより、カスタムレポート、特定のデータ入力フォーム、外部サービスとの連携インターフェースなど、Exmentの標準機能では提供されない独自のUIと機能を持つ画面を構築できます。

#### 作成方法

1.  **`Plugin.php`ファイルの作成**: `app/Plugins` ディレクトリ内に、プラグイン名に対応するディレクトリを作成し、その中に`Plugin.php`ファイルを作成します。
2.  **`config.json`の設定**: `config.json`ファイルで`plugin_type`を`page`に設定し、`route`セクションでページのURIと対応するコントローラメソッドを定義します。
3.  **ページロジックの実装**: `Plugin.php`内で、`PluginPageBase`を継承したクラスを作成し、`route`で定義したメソッドにページの表示ロジックを記述します。LaravelのBladeテンプレートエンジンを使用してUIを構築できます。

#### 例: カスタムレポートページ

```php
<?php

namespace App\Plugins\CustomReport;

use Exceedone\Exment\Services\Plugin\PluginPageBase;
use Exceedone\Exment\Model\CustomTable;

class Plugin extends PluginPageBase
{
    /**
     * config.jsonで定義されたURIに対応するメソッドです。
     *
     * @return \Illuminate\View\View
     */
    public function index()
    {
        // レポートデータを取得
        $data = CustomTable::find("your_table_id")
            ->data()
            ->get();

        // Bladeテンプレートを呼び出し
        return $this->pluginView("report", ["data" => $data]);
    }
}
```

#### `config.json`の例

```json
{
    "plugin_name": "CustomReport",
    "plugin_view_name": "カスタムレポート",
    "description": "独自のレポートを表示するページです。",
    "uuid": "your-unique-uuid-here",
    "author": "(Your Name)",
    "version": "1.0.0",
    "plugin_type": "page",
    "route": [
        {
            "uri": "/",
            "method": ["get"],
            "function": "index"
        }
    ]
}
```

#### 注意事項

*   ページプラグインは、特定のテーブルに紐付かない独立した画面を作成するのに適しています。
*   ルーティング設定により、複雑なURL構造やHTTPメソッドに対応できます。

### ドキュメント

ドキュメントプラグインは、Exmentのカスタムデータ詳細画面に、独自の情報を表示したり、既存の情報を加工して表示する機能を提供します。これにより、データの視覚化、関連情報の表示、外部システムからのデータ埋め込みなど、詳細画面の情報をより豊かにすることができます。

#### 作成方法

1.  **`Plugin.php`ファイルの作成**: `app/Plugins` ディレクトリ内に、プラグイン名に対応するディレクトリを作成し、その中に`Plugin.php`ファイルを作成します。
2.  **`config.json`の設定**: `config.json`ファイルで`plugin_type`を`document`に設定します。
3.  **ドキュメントロジックの実装**: `Plugin.php`内で、`PluginDocumentBase`を継承したクラスを作成し、`detail`メソッドに詳細画面に表示するロジックを記述します。`detail`メソッドは、カスタムデータの詳細画面が表示される際に呼び出されます。

#### 例: 関連するタスクを表示するドキュメントプラグイン

```php
<?php

namespace App\Plugins\RelatedTasks;

use Exceedone\Exment\Services\Plugin\PluginDocumentBase;
use Exceedone\Exment\Model\CustomTable;

class Plugin extends PluginDocumentBase
{
    /**
     * カスタムデータ詳細画面に表示する処理です。
     *
     * @param array $data 現在表示されているカスタムデータの情報
     * @return \Illuminate\View\View
     */
    public function detail($data)
    {
        // 現在のデータに関連するタスクを取得
        $relatedTasks = CustomTable::find("tasks_table_id")
            ->data()
            ->where("project_id", $data["id"])
            ->get();

        return $this->pluginView("related_tasks", ["tasks" => $relatedTasks]);
    }
}
```

#### 注意事項

*   ドキュメントプラグインは、カスタムデータの詳細画面のコンテキストで動作します。
*   `detail`メソッドには、現在表示されているカスタムデータの情報が`$data`として渡されます。

### ダッシュボード
基本的にダッシュボードにピッタリと収まる余白なく収まるようにすること。
ダッシュボードプラグインは、Exmentのトップページにカスタムウィジェットや情報表示を追加する機能を提供します。これにより、ユーザーはログイン時に重要な情報を一目で確認できます。

#### 基本的な作成方法

1.  **`Plugin.php`ファイルの作成**: `app/Plugins` ディレクトリ内に、プラグイン名に対応するディレクトリを作成し、その中に`Plugin.php`ファイルを作成します。
2.  **`config.json`の設定**: `config.json`ファイルで`plugin_type`を`dashboard`に設定します。
3.  **ダッシュボードウィジェットの実装**: `Plugin.php`内で、`PluginDashboardBase`を継承したクラスを作成し、`body`メソッドにダッシュボードウィジェットのロジックを記述します。

#### ダッシュボード時計プラグインの完全な実装例

以下は、リアルタイムで更新される時計を表示するダッシュボードプラグインの完全な実装例です。この例を参考にすることで、1回でピッタリと動作するダッシュボードプラグインを作成できます。

##### `config.json`

```json
{
    "plugin_name": "ExmentClock",
    "plugin_view_name": "Exmentダッシュボード時計",
    "description": "ダッシュボードにリアルタイム時計を表示します",
    "uuid": "12345678-1234-1234-1234-123456789abc",
    "author": "Exment開発チーム",
    "version": "1.0.0",
    "plugin_type": "dashboard"
}
```

**重要なポイント:**
- `plugin_type`は必ず`dashboard`に設定
- `uuid`は一意の値を生成（[UUIDジェネレーター](https://www.uuidgenerator.net/)を使用）
- `plugin_name`はディレクトリ名と一致させる

##### `Plugin.php`

```php
<?php

namespace App\Plugins\ExmentClock;

use Exceedone\Exment\Services\Plugin\PluginDashboardBase;
use Carbon\Carbon;

class Plugin extends PluginDashboardBase
{
    /**
     * ダッシュボードの本体を返す
     * 
     * @return string HTML文字列
     */
    public function body()
    {
        // 現在の日本時間を取得
        $now = Carbon::now('Asia/Tokyo');
        
        // HTML文字列として時計を生成
        return $this->generateClockHtml($now);
    }
    
    /**
     * 時計のHTML文字列を生成
     * 
     * @param Carbon $time 現在時刻
     * @return string
     */
    private function generateClockHtml($time)
    {
        $formattedDate = $time->format('Y年m月d日');
        $formattedTime = $time->format('H:i:s');
        $dayOfWeek = ['日', '月', '火', '水', '木', '金', '土'][$time->dayOfWeek];
        
        return '
        <div style="
            background: #fff; 
            padding: 20px; 
            border-radius: 3px; 
            box-shadow: 0 1px 1px rgba(0,0,0,0.1); 
            margin: 0;
            text-align: center; 
            font-family: \'Arial\', sans-serif;
        ">
            <div style="font-size: 18px; color: #333; margin-bottom: 10px; font-weight: bold;">
                📅 ' . $formattedDate . ' (' . $dayOfWeek . ')
            </div>
            <div id="current-time" style="
                font-size: 32px; 
                color: #007bff; 
                font-weight: bold; 
                font-family: \'Courier New\', monospace;
                text-shadow: 1px 1px 2px rgba(0,0,0,0.1);
            ">
                🕐 ' . $formattedTime . '
            </div>
            <div style="
                font-size: 12px; 
                color: #666; 
                margin-top: 8px;
            ">
                日本標準時 (JST)
            </div>
        </div>

        <script>
        (function() {
            function updateClock() {
                const now = new Date();
                const jst = new Date(now.getTime() + (9 * 60 * 60 * 1000));
                const timeString = jst.toTimeString().split(\' \')[0];
                const timeElement = document.getElementById(\'current-time\');
                if (timeElement) {
                    timeElement.innerHTML = \'🕐 \' + timeString;
                }
            }
            
            // 1秒ごとに時刻を更新
            setInterval(updateClock, 1000);
            
            // 初回実行
            updateClock();
        })();
        </script>
        ';
    }
}
```

#### ダッシュボードプラグイン開発のベストプラクティス

##### 1. スタイリングの注意点

- **既存プラグインとの統一**: Exmentの他のダッシュボードプラグインと同様のスタイルを適用
- **レスポンシブ対応**: 様々な画面サイズで適切に表示されるように設計
- **マージンの調整**: `margin: 0`を設定して、余分な空白を削除

##### 2. JavaScriptの実装

```javascript
// 即座実行関数でスコープを分離
(function() {
    function updateFunction() {
        // 更新処理
    }
    
    // インターバル設定
    setInterval(updateFunction, 1000);
    
    // 初回実行
    updateFunction();
})();
```

##### 3. 日時処理での注意

```php
// Carbonを使用して日本時間を正確に取得
$now = Carbon::now('Asia/Tokyo');

// JavaScriptでもJSTを考慮
const jst = new Date(now.getTime() + (9 * 60 * 60 * 1000));
```

##### 4. HTMLとCSSのインライン記述

ダッシュボードプラグインでは、通常はBladeビューファイルを使用せず、PHP内でHTML文字列を直接生成します。これにより、ビューのnamespace問題を回避できます。

#### よくあるエラーとその解決方法

##### 1. ビューファイル関連のエラー

**エラー例**: `No hint path defined for [plugin_name]`

**解決方法**: Bladeビューファイルを使用せず、HTML文字列を直接返すように実装する。

##### 2. スタイルの問題

**問題**: ダッシュボードに青い線や余分なマージンが表示される

**解決方法**: 
```css
margin: 0;
border: none;
box-shadow: 0 1px 1px rgba(0,0,0,0.1);
```

##### 3. JavaScript実行タイミングの問題

**解決方法**: 即座実行関数を使用してDOMロード完了を待つ必要なく実行

#### 高度な実装例: データベース連携

```php
public function body()
{
    // カスタムテーブルからデータを取得
    $taskCount = CustomTable::getEloquent('tasks')
        ->getValueModel()
        ->where('value->status', '進行中')
        ->count();
    
    return $this->generateTaskCountHtml($taskCount);
}

private function generateTaskCountHtml($count)
{
    return '
    <div style="background: #fff; padding: 20px; border-radius: 3px; box-shadow: 0 1px 1px rgba(0,0,0,0.1); margin: 0; text-align: center;">
        <h3 style="color: #333; margin: 0 0 10px 0;">📋 進行中タスク</h3>
        <div style="font-size: 48px; color: #28a745; font-weight: bold;">' . $count . '</div>
        <div style="color: #666; font-size: 14px;">件のタスクが進行中です</div>
    </div>';
}
```

#### プラグインのパッケージ化

1. **ディレクトリ構成**:
```
ExmentClock/
├── config.json
└── Plugin.php
```

2. **ZIPファイル作成**: 
- ファイル名: `ExmentClock.zip`
- 最小構成でパッケージ化

#### デプロイと動作確認

1. **プラグインのアップロード**: Exment管理画面 > プラグイン > 新規追加
2. **ダッシュボードでの確認**: トップページでウィジェットが正しく表示されることを確認
3. **リアルタイム更新の確認**: 時刻が1秒ごとに更新されることを確認

この実装例を参考にすることで、Exmentのダッシュボードプラグインを確実に作成し、デプロイすることができます。

#### 注意事項

*   ダッシュボードウィジェットは、Laravel AdminのBoxコンポーネントの代わりにHTML文字列を直接返すことで、より柔軟な表示が可能です。
*   リアルタイム更新が必要な場合は、JavaScriptのsetIntervalを活用してください。
*   スタイリングは既存のExmentプラグインと統一感を保つようにしてください。

### バッチ

バッチプラグインは、定期的に実行される処理や、大量のデータを処理するタスクを定義します。これにより、データの一括更新、外部システムとの連携、レポート生成などを自動化できます。

#### 作成方法

1.  **`Plugin.php`ファイルの作成**: `app/Plugins` ディレクトリ内に、プラグイン名に対応するディレクトリを作成し、その中に`Plugin.php`ファイルを作成します。
2.  **`config.json`の設定**: `config.json`ファイルで`plugin_type`を`batch`に設定します。
3.  **バッチロジックの実装**: `Plugin.php`内で、`PluginBatchBase`を継承したクラスを作成し、`handle`メソッドにバッチ処理のロジックを記述します。

#### 例: 定期的にデータを集計するバッチプラグイン

```php
<?php

namespace App\Plugins\YourBatchPlugin;

use Exceedone\Exment\Services\Plugin\PluginBatchBase;
use Exceedone\Exment\Model\CustomTable;

class Plugin extends PluginBatchBase
{
    /**
     * バッチ処理を実行する際に呼び出される処理です。
     *
     * @return void
     */
    public function handle()
    {
        // ここにバッチ処理のロジックを記述
        // 例: 特定のテーブルのデータを集計し、別のテーブルに保存
        $sourceTable = CustomTable::find("source_table_id");
        $destinationTable = CustomTable::find("destination_table_id");

        $dataToProcess = $sourceTable->data()->get();

        foreach ($dataToProcess as $data) {
            // 処理ロジック
            $processedData = [
                "column1" => $data->column1,
                "column2" => $data->column2 * 2,
            ];
            $destinationTable->data()->create($processedData);
        }
    }
}
```

#### 注意事項

*   バッチプラグインは、Exmentのタスクスケジューラと連携して、指定した時間に自動実行できます。
*   大量のデータを処理する場合は、メモリ使用量や実行時間に注意し、必要に応じて処理を分割することを検討してください。

### スクリプト

スクリプトプラグインは、Exmentの画面にJavaScriptを追加する機能を提供します。これにより、ページの動作をカスタマイズしたり、インタラクティブな機能を追加できます。

#### 作成方法

1.  **JavaScriptファイルの作成**: プラグインディレクトリ内に、`.js`ファイルを作成します。
2.  **`config.json`の設定**: `config.json`ファイルで`plugin_type`を`script`に設定します。
3.  **スクリプトの実装**: JavaScriptファイル内に、必要な機能を記述します。

#### 例: ページにカスタム機能を追加するスクリプト

```javascript
// custom_script.js
$(document).ready(function() {
    // ページ読み込み完了時に実行される処理
    console.log("Exmentカスタムスクリプトが読み込まれました");
    
    // カスタムボタンを追加
    $(\'body\').append(\'<button id="custom-button">カスタム機能</button>\');
    
    // ボタンクリック時の処理
    $(\'#custom-button\').click(function() {
        alert(\'カスタム機能が実行されました！\');
    });
});
```

#### `config.json`の例

```json
{
    "plugin_name": "CustomScript",
    "plugin_view_name": "カスタムスクリプト",
    "description": "ページにカスタム機能を追加します。",
    "uuid": "your-unique-uuid-here",
    "author": "(Your Name)",
    "version": "1.0.0",
    "plugin_type": "script"
}
```

#### 注意事項

*   スクリプトプラグインは、Exmentの全ページで読み込まれます。
*   jQueryやBootstrapなど、Exmentで使用されているライブラリを活用できます。
*   セキュリティに配慮し、信頼できるスクリプトのみを使用してください。

### スタイル

スタイルプラグインは、Exmentの画面にカスタムCSSを追加する機能を提供します。これにより、ExmentのUIを自由にカスタマイズし、ブランドイメージに合わせることができます。

#### 作成方法

1.  **CSSファイルの作成**: プラグインディレクトリ内に、`.css`ファイルを作成します。
2.  **`config.json`の設定**: `config.json`ファイルで`plugin_type`を`style`に設定します。
3.  **スタイルの実装**: CSSファイル内に、必要なスタイルを記述します。

#### 例: Exmentのヘッダーの色を変更するスタイル

```css
/* custom_style.css */
.main-header {
    background-color: #FF5733 !important; /* 例: オレンジ色に変更 */
}

.main-header .logo {
    background-color: #C70039 !important; /* 例: ロゴの背景色も変更 */
}
```

#### `config.json`の例

```json
{
    "plugin_name": "CustomStyle",
    "plugin_view_name": "カスタムスタイル",
    "description": "ExmentのUIをカスタマイズします。",
    "uuid": "your-unique-uuid-here",
    "author": "(Your Name)",
    "version": "1.0.0",
    "plugin_type": "style"
}
```

#### 注意事項

*   スタイルプラグインは、Exmentの全ページで読み込まれます。
*   既存のスタイルを上書きする場合は、`!important`を使用する必要がある場合があります。
*   変更が意図しない表示崩れを引き起こさないよう、慎重にスタイルを適用してください。

## その他の特殊設定

### プラグイン設定画面で独自の設定を行う

プラグインに独自のカスタム設定画面を追加することで、ユーザーがExmentの管理画面からプラグインの動作を細かく調整できるようになります。これにより、プラグインの汎用性と使いやすさが向上します。

#### 設定方法

1.  **`Plugin.php`の修正**: `Plugin.php`ファイル内で、`protected $useCustomOption = true;` を追加します。
2.  **`setCustomOptionForm`メソッドの実装**: `Plugin.php`内で、`public function setCustomOptionForm(&$form)` メソッドを実装し、設定画面に表示するフォーム要素を定義します。Laravel Adminのフォームコンポーネントを利用できます。
3.  **設定値の取得**: プラグインのロジック内で、`$this->plugin->getCustomOption(\'parameter_name\')` を使用して、設定画面で入力された値を取得します。

#### 例: YouTube Data APIのアクセスキーを設定するプラグイン

```php
<?php

namespace App\Plugins\YouTubeSearch;

use Encore\Admin\Widgets\Box;
use Exceedone\Exment\Model\CustomTable;
use Exceedone\Exment\Services\Plugin\PluginPageBase;
use GuzzleHttp\Client;

class Plugin extends PluginPageBase
{
    protected $useCustomOption = true; // (1) カスタムオプションを有効にする

    /**
     * (2) プラグイン編集画面で設定するオプション
     *
     * @param $form
     * @return void
     */
    public function setCustomOptionForm(&$form)
    {
        $form->text(\'access_key\', \'アクセスキー\')
            ->help(\'YouTubeのアクセスキーを入力してください。\');
    }

    // ... その他のプラグインロジック ...

    public function someFunction()
    {
        $accessKey = $this->plugin->getCustomOption(\'access_key\'); // 設定値の取得
        // ... アクセスキーを使用した処理 ...
    }
}
```

#### 注意事項

*   機密性の高い情報は、環境変数やExmentのシステム設定を利用することも検討してください。
*   フォーム要素のバリデーションを適切に行い、不正な入力からプラグインを保護してください。

### 1つのプラグインで複数のプラグインタイプをサポートする

通常、1つのプラグインファイルは1つのプラグインタイプのみをサポートします。しかし、Exmentでは、特殊な設定を行うことで、1つのプラグインファイルで複数のプラグインタイプ（例: ページとダッシュボード）をサポートすることが可能です。これにより、関連する機能をまとめて管理し、開発効率を向上させることができます。

#### 設定方法

1.  **`config.json`の変更**: `config.json`ファイルの`plugin_type`を、カンマ区切りで複数のプラグインタイプを指定します。
    ```json
    // config.json 抜粋
    "plugin_type": "dashboard,page"
    ```
2.  **PHPファイルの変更**: 各プラグインタイプに対応するPHPファイルを作成し、ファイル名を`Plugin`の後にプラグインタイプ名を付けた形（例: `PluginPage.php`, `PluginDashboard.php`）に変更します。また、クラス名もファイル名に合わせて変更します。
    ```php
    <?php
    
    // クラス名の変更 (トリガーの場合)
    class PluginTrigger extends PluginPageBase
    {
        // ...
    }
    ```
3.  **(オプション) 設定用PHPファイルの作成**: プラグイン設定画面で独自の設定を行いたい場合は、`PluginSetting.php`ファイルを作成し、`PluginSettingBase`を継承したクラスを定義します。

#### 圧縮とデプロイ

これらのファイルを最小構成でzipファイルに圧縮します。zipファイル名は「(プラグイン名).zip」とします。

*   XXXX.zip
    *   config.json
    *   PluginDashboard.php
    *   PluginPage.php
    *   PluginSetting.php (オプション)
    *   (その他の必要なPHPファイル、画像ファイルなど)

#### 注意事項

*   `Script`または`Style`タイプのプラグインの場合、PHPファイルの変更は不要です。
*   各プラグインタイプに対応するPHPファイルは、それぞれの`PluginTypeBase`を継承する必要があります。
*   プラグインのデプロイには、Exmentの管理画面からzipファイルをアップロードします。

## サンプルプラグイン

Exmentの公式GitHubリポジトリには、様々なプラグインタイプのサンプルコードが提供されています[2]。これらのサンプルは、実際のプラグイン開発の参考として非常に有用です。

## プラグインのデプロイ

開発したプラグインは、以下の手順でExmentにデプロイします。

1.  プラグインのファイルをZIP形式で圧縮します。`config.json`と必要なPHPファイル、アセットファイルを含めます。
2.  Exmentの管理画面にログインします。
3.  「プラグイン」セクションに移動し、「プラグインの追加」または類似のオプションを選択します。
4.  作成したZIPファイルをアップロードします。
5.  プラグインが正常にインストールされたら、必要に応じて設定を行います。

### APIプラグインのサンプル: PluginDemoAPI

**概要**: `PluginDemoAPI`は、Exmentのカスタム列情報をAPI経由で取得するサンプルプラグインです。特定のテーブルと列名を指定することで、その列の詳細情報をJSON形式で返します。これは、外部システムからExmentのメタデータにアクセスする際に役立ちます。

**`config.json`の要約**:

```json
{
    "plugin_name": "PluginDemoAPI",
    "uuid": "50716af0-05d3-11ea-aaef-0800200c9a66",
    "plugin_view_name": "API Plugin Sample",
    "description": "列名からカスタム列情報を取得するサンプルAPIです。",
    "author": "Kajitori",
    "version": "1.0.0",
    "plugin_type": "api",
    "uri": "sampleapi",
    "route": [
        {
            "uri": "column",
            "method": [
                "get"
            ],
            "function": "column"
        },
        {
            "uri": "tablecolumn/{table}/{column}",
            "method": [
                "get"
            ],
            "function": "tablecolumn"
        }
    ]
}
```

*   `plugin_type`: `api`として定義されており、APIプラグインであることを示します。
*   `uri`: このAPIプラグインのベースURIが`sampleapi`であることを示します。
*   `route`: 2つのエンドポイントが定義されています。
    *   `/column`: GETリクエストで`table`と`column`パラメータを受け取り、`column`メソッドを実行します。
    *   `/tablecolumn/{table}/{column}`: GETリクエストでURLパスから`table`と`column`を受け取り、`tablecolumn`メソッドを実行します。

**`Plugin.php`の要約**:

```php
<?php
namespace App\Plugins\PluginDemoAPI;

use Exceedone\Exment\Enums\ErrorCode;
use Exceedone\Exment\Enums\Permission;
use Exceedone\Exment\Model\CustomTable;
use Exceedone\Exment\Services\Plugin\PluginApiBase;
use Validator;

class Plugin extends PluginApiBase
{
    /**
     * カラム名からカスタム列情報を取得するサンプルです
     * @return mixed
     */
    public function column()
    {
        // リクエストパラメータをチェックします（tableとcolumnはそれぞれ必須）
        $validator = Validator::make(request()->all(), [
            \'table\' => \'required\',
            \'column\' => \'required\',
        ]);
        if ($validator->fails()) {
            return abortJson(400, [
                \'errors\' => $this->getErrorMessages($validator)
            ], ErrorCode::VALIDATION_ERROR());
        }

        // リクエストパラメータからテーブル名とカラム名を取得します
        $table_name = request()->get(\'table\');
        $column_name = request()->get(\'column\');

        // カスタムテーブル情報を取得します
        $custom_table = CustomTable::where(\'table_name\', $table_name)->first();
        if (!isset($custom_table)) {
            return abort(400);
        }

        // 権限があるかチェックします
        if (!$custom_table->hasPermission(Permission::AVAILABLE_ACCESS_CUSTOM_VALUE)) {
            return abortJson(403, trans(\'admin.deny\'));
        }

        // カスタム列情報を取得します
        $column = $custom_table->custom_columns()->where(\'column_name\', $column_name)->first();
        if (!isset($column)) {
            return abort(400);
        }

        return $column;
    }

    /**
     * カラム名からカスタム列情報を取得するサンプルです
     * ※URLでテーブル名とカラム名を指定しています
     * @return mixed
     */
    public function tablecolumn($table, $column)
    {
        // カスタムテーブル情報を取得します
        $custom_table = CustomTable::where(\'table_name\', $table)->first();
        if (!isset($custom_table)) {
            return abort(400);
        }

        // 権限があるかチェックします
        if (!$custom_table->hasPermission(Permission::AVAILABLE_ACCESS_CUSTOM_VALUE)) {
            return abortJson(403, trans(\'admin.deny\'));
        }

        // カスタム列情報を取得します
        $custom_column = $custom_table->custom_columns()->where(\'column_name\', $column)->first();
        if (!isset($custom_column)) {
            return abort(400);
        }

        return $custom_column;
    }
}
```

*   `PluginApiBase`を継承しており、APIプラグインの基底機能を提供します。
*   `column()`メソッド:
    *   リクエストパラメータ`table`と`column`のバリデーションを行います。
    *   指定されたテーブル名と列名に基づいて、`CustomTable`および`CustomColumn`モデルから情報を取得します。
    *   ユーザーが対象のカスタム値にアクセスする権限を持っているかを確認します。
    *   取得したカスタム列情報を返します。
*   `tablecolumn($table, $column)`メソッド:
    *   URLパスから直接`table`と`column`を受け取ります。
    *   `column()`メソッドと同様に、テーブル情報の取得、権限チェック、カスタム列情報の取得を行い、結果を返します。

**このプラグインから学べること**:

*   ExmentにおけるAPIプラグインの基本的な構造と`config.json`の設定方法。
*   `PluginApiBase`を継承したクラスでのAPIエンドポイントの実装方法。
*   リクエストパラメータのバリデーション、データベースからのデータ取得（`CustomTable`, `CustomColumn`モデルの使用）、および権限チェックの実施方法。
*   RESTfulなAPI設計の基本的な考え方（パスパラメータの使用など）。




### バッチプラグインのサンプル: HarddeleteBatch

**概要**: `HarddeleteBatch`は、Exment内で論理削除されたすべてのデータを物理的に削除するバッチプラグインです。定期的に実行することで、データベースのクリーンアップとパフォーマンス維持に貢献します。

**`config.json`の要約**:

```json
{
    "uuid": "b5c0a5d2-2716-4161-98d0-b490c1ebc521",
    "plugin_name": "harddelete_data",
    "plugin_view_name": "データの物理削除",
    "description": "論理削除しているすべてのデータを、完全に削除します。",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "batch",
    "batch_hour": 3,
    "batch_cron": "10,20,30 * * * *"
}
```

*   `plugin_type`: `batch`として定義されており、バッチプラグインであることを示します。
*   `batch_hour`: バッチが実行される時間（この場合は3時）を示します。
*   `batch_cron`: バッチの実行スケジュールをCron形式で定義しています。この例では、毎時10分、20分、30分に実行されます。

**`Plugin.php`の要約**:

```php
<?php
namespace App\Plugins\HarddeleteData;

use Exceedone\Exment\Services\Plugin\PluginBatchBase;
use Exceedone\Exment\Model\CustomTable;

class Plugin extends PluginBatchBase{
    /**
     * execute
     */
    public function execute() {
        $tables = CustomTable::all();

        foreach($tables as $table){
            $modelname = getModelName($table);
            if(!isset($modelname)){
                continue;
            }

            $modelname::onlyTrashed()
                ->forceDelete();
        }
    }
    
}
```

*   `PluginBatchBase`を継承しており、バッチプラグインの基底機能を提供します。
*   `execute()`メソッド:
    *   Exment内のすべてのカスタムテーブルを取得します。
    *   各テーブルに対して、論理削除されたレコード（`onlyTrashed()`）を物理的に削除（`forceDelete()`）します。

**このプラグインから学べること**:

*   Exmentにおけるバッチプラグインの基本的な構造と`config.json`でのスケジュール設定方法。
*   `PluginBatchBase`を継承したクラスでのバッチ処理の実装方法。
*   `CustomTable`モデルを使用してExmentのテーブル情報にアクセスし、論理削除されたデータを操作する方法。




### ボタンプラグインのサンプル: PluginCustomButton

**概要**: `PluginCustomButton`は、Exmentのデータ詳細画面に「次へ」「前へ」ボタンを追加するプラグインです。これにより、ユーザーはレコード間を簡単に移動できるようになります。

**`config.json`の要約**:

```json
{
    "plugin_name": "PluginCustomButton",
    "uuid": "6f0777c0-8e36-11ea-ab12-0800200c9a12",
    "plugin_view_name": "Plugin Custom Button",
    "description": "独自のプラグインボタンを追加するテストです。",
    "author": "(Your Name)",
    "version": "1.0.0",
    "plugin_type": "button",
    "event_triggers": "form_menubutton_show",
    "target_tables": "information"
}
```

*   `plugin_type`: `button`として定義されており、ボタンプラグインであることを示します。
*   `event_triggers`: `form_menubutton_show`に設定されており、フォームのメニューボタンが表示されるイベントでこのプラグインがトリガーされることを示します。
*   `target_tables`: `information`に設定されており、`information`テーブルにのみこのボタンが表示されることを示します。

**`Plugin.php`の要約**:

```php
<?php
namespace App\Plugins\PluginCustomButton;

use Exceedone\Exment\Services\Plugin\PluginButtonBase;
class Plugin extends PluginButtonBase
{
    /**
     * Plugin Trigger
     */
    public function execute()
    {
    }
    
    /**
     * デフォルトのボタン表示に代わり、独自のボタンを追加する処理
     *
     * @return void
     */
    public function render(){
        $buttons = [];

        // keys. true: 「次」、false：「前」
        $keys = [true, false];
        foreach($keys as $key){
            // 前後の値を取得
            $value = $this->getNextOrPrevId($key);
            
            // 値が存在すれば、ボタンを追加する
            if(!is_null($value)){
                $buttons[] = [
                    \"href\" => $value->getUrl(),
                    \"target\" => \"_top\",


                    ($key ? 'icon_right' : 'icon') => $key ? 'fa-arrow-right' : 'fa-arrow-left',
                    'label' => $value->getLabel(),
                ];
            }
        }

        // button.blade.phpでボタンを描写する
        return $this->pluginView('buttons', ['buttons' => $buttons]);
    }

    /**
     * 次もしくは前のデータを取得する
     *
     * @param boolean $isNext true: 次のデータ、false: 前のデータ
     * @return CustomValue
     */
    protected function getNextOrPrevId(bool $isNext){
        $query = $this->custom_table->getValueModel()->query();   
        $query->where('id', ($isNext ? '>' : '<'), $this->custom_value->id);

        $query->orderBy('id', ($isNext ? 'asc' : 'desc'));

        return $query->first();
    }
}
```

*   `PluginButtonBase`を継承しており、ボタンプラグインの基底機能を提供します。
*   `execute()`メソッド: このサンプルでは特に処理は記述されていません。
*   `render()`メソッド:
    *   `getNextOrPrevId()`メソッドを使用して、現在のレコードの「次」と「前」のレコードIDを取得します。
    *   取得したレコードIDに基づいて、ExmentのURLを生成し、ボタンのリンクとして設定します。
    *   `pluginView('buttons', ['buttons' => $buttons])`を呼び出し、`buttons.blade.php`テンプレートにボタン情報を渡して描画します。
*   `getNextOrPrevId(bool $isNext)`メソッド:
    *   現在のカスタムテーブルのモデルからクエリを構築します。
    *   現在のレコードIDを基準に、次のレコード（`id > current_id`かつ`asc`ソート）または前のレコード（`id < current_id`かつ`desc`ソート）を取得します。

**このプラグインから学べること**:

*   Exmentにおけるボタンプラグインの基本的な構造と`config.json`でのイベントトリガー、対象テーブルの設定方法。
*   `PluginButtonBase`を継承したクラスでのボタンの表示ロジックの実装方法。
*   `render()`メソッドを使用してカスタムボタンを生成し、Bladeテンプレートで描画する方法。
*   `CustomTable`モデルとクエリビルダを使用して、Exmentのデータにアクセスし、レコード間を移動するロジックを実装する方法。




### CRUDプラグインのサンプル: MySQLWorld

**概要**: `MySQLWorld`は、Exmentとは異なる外部データベース（MySQLのサンプルデータベース「World」）と連携し、そのデータベース内の都市データに対してCRUD（作成、読み取り、更新、削除）操作を提供するCRUDプラグインです。Exmentの管理画面から外部データを直接操作できる機能を提供します。

**`config.json`の要約**:

```json
{
    "plugin_name": "MySQLWorld",
    "plugin_view_name" : "MySQL World連携",
    "description": "MySQLのサンプルデータベース「World」とExmentを連携します。",
    "uuid":  "2641201a-ba35-2bd9-af59-9440643ca206",
    "author":  "Kajitori",
    "version": "1.0.0",
    "uri":  "mysql_world",
    "plugin_type": "crud"
}
```

*   `plugin_type`: `crud`として定義されており、CRUDプラグインであることを示します。
*   `uri`: このCRUDプラグインのベースURIが`mysql_world`であることを示します。

**`Plugin.php`の要約**:

```php
<?php

namespace App\Plugins\MySQLWorld;

use Encore\Admin\Widgets\Grid\Grid;
use Encore\Admin\Widgets\Form;
use Exceedone\Exment\Services\Plugin\PluginCrudBase;
use Illuminate\Support\Collection;
use Illuminate\Pagination\LengthAwarePaginator;

class Plugin extends PluginCrudBase
{
    protected $title = \'世界の都市\';
    protected $description = \'世界の都市を一覧で表示します。\';
    protected $icon = \'fa-globe\';

    /**
     * Get fields definitions
     *
     * @return array|Collection
     */
    public function getFieldDefinitions()
    {
        return [
            [\'key\' => \'ID\', \'label\' => \'ID\', \'primary\' => true, \'grid\' => 1, \'show\' => 1, \'edit\' => 1],
            [\'key\' => \'Name\', \'label\' => \'都市名\', \'grid\' => 2,\'show\' => 2, \'create\' => 1, \'edit\' => 2],
            [\'key\' => \'CountryCode\', \'label\' => \'国コード\', \'grid\' => 3, \'show\' => 3, \'create\' => 2,\'edit\' => 3],
            [\'key\' => \'Population\', \'label\' => \'人工\', \'show\' => 5, \'create\' => 4,\'edit\' => 5],
        ];
    }

    /**
     * Get data paginate
     *
     * @return LengthAwarePaginator
     */
    public function getPaginate(array $options = []) : ?LengthAwarePaginator
    {
        $query = \DB::connection(\'world\')
            ->table(\'city\');

        // フリーワード検索がある場合
        $q = array_get($options, \'query\');
        if(isset($q)){
            $query->where(function($query) use($q){
                $query
                    ->where(\'Name\', \'LIKE\', "%{\$q}%")
                    ->orWhere(\'CountryCode\', \'LIKE\', "%{\$q}%")
                ;
            });
        }

        return $query->paginate(array_get($options, \'per_page\') ?? 20, [\'*\'], \'page\', array_get($options, \'page\'));
    }

    /**
     * read single data
     *
     * @return array|Collection
     */
    public function getData($id, array $options = [])
    {
        return \DB::connection(\'world\')
            ->table(\'city\')
            ->where(\'ID\', $id)->first();
    }

    /**
     * set form info
     *
     * @return Form|null
     */
    public function setForm(Form $form, bool $isCreate, array $options = []) : ?Form
    {
        if(!$isCreate){
            $form->display(\'ID\');    
        }
        $form->text(\'Name\');

        // 国一覧取得
        $countries = \DB::connection(\'world\')->table(\'country\')->pluck(\'Name\', \'Code\');
        $form->select(\'CountryCode\')->options($countries);
        
        $form->number(\'Population\');

        return $form;
    }

    /**
     * post create value
     *
     * @return mixed
     */
    public function postCreate(array $posts, array $options = [])
    {
        // 独自のデータベースに保存する。
        $value = \DB::connection(\'world\')
            ->table(\'city\')
            ->insertGetId($posts);

        return $value;
    }

    /**
     * edit posted value
     *
     * @return mixed
     */
    public function putEdit($id, array $posts, array $options = [])
    {
        // 独自のデータベースに保存する。
        \DB::connection(\'world\')
            ->table(\'city\')
            ->whereOrIn(\'ID\', $id)
            ->update($posts);

        return $id;
    }

    /**
     * delete value
     *
     * @param $id string
     * @return mixed
     */
    public function delete($id, array $options = [])
    {
        $value = \DB::connection(\'world\')
            ->table(\'city\')
            ->where(\'ID\', $id)
            ->delete();
    }
}
```

*   `PluginCrudBase`を継承しており、CRUDプラグインの基底機能を提供します。
*   `$title`, `$description`, `$icon`: 管理画面での表示名、説明、アイコンを定義します。
*   `getFieldDefinitions()`: CRUD操作の対象となるフィールド（列）の定義を返します。各フィールドのキー、ラベル、表示/編集/作成時の設定が含まれます。
*   `getPaginate()`: 外部データベースからデータをページネーションして取得します。フリーワード検索にも対応しています。
*   `getData($id)`: 指定されたIDの単一のデータを外部データベースから取得します。
*   `setForm()`: データ作成/編集フォームのフィールドを定義します。ここでは、都市名、国コード、人口などの入力フィールドが設定されています。国コードは外部データベースから取得した国一覧から選択できるようにしています。
*   `postCreate()`: 新しいデータが作成された際に、外部データベースに挿入します。
*   `putEdit()`: 既存のデータが更新された際に、外部データベースのレコードを更新します。
*   `delete()`: データが削除された際に、外部データベースのレコードを削除します。

**このプラグインから学べること**:

*   ExmentにおけるCRUDプラグインの基本的な構造と`config.json`の設定方法。
*   `PluginCrudBase`を継承したクラスでのCRUD操作の実装方法。
*   `getFieldDefinitions()`、`getPaginate()`、`getData()`、`setForm()`、`postCreate()`、`putEdit()`、`delete()`といったCRUDプラグインの主要なメソッドの役割と実装例。
*   Exmentから外部データベースに接続し、データを操作する方法（`DB::connection()`の使用）。
*   Laravel Adminのフォームコンポーネント（`$form->text()`, `$form->select()`, `$form->number()`など）を使用して、データ入力フォームを構築する方法。




### ダッシュボードプラグインのサンプル: Dakoku

**概要**: `Dakoku`は、Exmentのダッシュボードに打刻機能を提供するプラグインです。ユーザーはダッシュボードから直接、出勤、休憩開始、休憩終了、退勤の操作を行うことができます。

**`config.json`の要約**:

```json
{
    "plugin_name": "Dakoku",
    "plugin_view_name" : "打刻",
    "explain": "ダッシュボードで打刻を行うためのプラグインです。",
    "uuid":  "691a24f2-2c7a-42c5-8cff-23c5277f6f22",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "dashboard",
    "route": [
        {
            "uri": "post",
            "method": [
                "post"
            ],
            "function": "post"
        }
    ]
}
```

*   `plugin_type`: `dashboard`として定義されており、ダッシュボードプラグインであることを示します。
*   `route`: `post`というURIでPOSTリクエストを受け付け、`post`メソッドを実行するルートが定義されています。これは打刻データの保存に使用されます。

**`Plugin.php`の要約**:

```php
<?php

namespace App\Plugins\Dakoku;

use Exceedone\Exment\Services\Plugin\PluginDashboardBase;
use Exceedone\Exment\Model\CustomTable;

class Plugin extends PluginDashboardBase
{
    /**
     *
     * @return Content|\Illuminate\Http\Response
     */
    public function body()
    {
        if(is_null(CustomTable::getEloquent(\'dakoku\'))){ // \'dakoku\'テーブルが存在しない場合のチェック
            return \'打刻テーブルがインストールされていません。\';
        }
        
        $dakoku = $this->getDakoku(); // 今日の打刻データを取得
        $status = $this->getStatus($dakoku); // 現在の打刻ステータスを取得
        $params = $this->getStatusParams($status); // ステータスに応じた表示パラメータを取得

        return view(\'exment_dakoku::dakoku\', [
            \'dakoku\' => $dakoku,
            \'params\' => $params,
            \'action\' => admin_url($this->getDashboardUri(\'post\')),
        ]);
    }

    /**
     * 送信
     *
     * @return void
     */
    public function post()
    {
        $dakoku = $this->getDakoku();
        
        $now = \Carbon\Carbon::now();

        if(!isset($dakoku)){ // 今日の打刻データがなければ新規作成
            $dakoku = CustomTable::getEloquent(\'dakoku\')->getValueModel();
            $dakoku->setValue(\'target_date\', $now->toDateString());
        }

        $status = null;
        switch(request()->get(\'action\')){
            // 出勤
            case \'syukkin\':
                $dakoku->setValue(\'syukkin_time\', $now);
                $status = 1;
                break;
            // 休憩開始
            case \'kyuukei_start\':
                $dakoku->setValue(\'kyuukei_start_time\', $now);
                $status = 11;
                break;
            // 休憩終了
            case \'kyuukei_end\':
                $dakoku->setValue(\'kyuukei_end_time\', $now);
                $status = 21;
                break;
            // 退勤
            case \'taikin\':
                $dakoku->setValue(\'taikin_time\', $now);
                $status = 99;
                break;
        }
        if(isset($status)){
            $dakoku->setValue(\'status\', $status);
        }
        $dakoku->save(); // 打刻データを保存

        admin_toastr(trans(\'admin.save_succeeded\'));
        return back();
    }

    /**
     * 現在の勤怠状況を取得
     *
     * @return void
     */
    protected function getDakoku(){
        $table = CustomTable::getEloquent(\'dakoku\');
        if(!isset($table)){
            return null;
        }
        
        
        // 現在時刻を取得
        $now = \Carbon\Carbon::now();

        // 基準時間より前であれば、前日として扱う
        if($now->hour <= 4){
            $now = $now->addDay(-1);
        }

        // 該当の打刻を取得
        $query = getModelName(\'dakoku\')::query();
        $query->where(\'value->target_date\', $now->toDateString())
            ->where(\'created_user_id\', \Exment::user()->base_user_id);

        $dakoku = $query->first();

        return $dakoku;
    }

    /**
     * ステータスを取得
     *
     * @param [type] $dakoku
     * @return int
     */
    protected function getStatus($dakoku){
        if(!isset($dakoku) || is_null($dakoku->getValue(\'syukkin_time\'))){
            return 0;
        }

        return $dakoku->getValue(\'status\');
    }

    protected function getStatusParams($status){
        switch($status){
            case 0:
                return 
                [
                    \'status_text\' => \'未出勤\',
                    \'buttons\' => [
                        [
                        \'button_text\' => \'出勤\',
                        \'action_name\' => \'syukkin\',
                        ]
                    ]
                ];
            // 出勤
            case 1:
                return  
                [
                    \'status_text\' => \'出勤中\',
                    \'buttons\' => [
                        [
                            \'button_text\' => \'休憩\',
                            \'action_name\' => \'kyuukei_start\',
                        ], 
                        [
                            \'button_text\' => \'退勤\',
                            \'action_name\' => \'taikin\',
                        ], 
                    ]
                ];
            case 11:
                return 
                [
                    \'status_text\' => \'休憩中\',
                    \'buttons\' => [
                        [
                            \'button_text\' => \'休憩終了\',
                            \'action_name\' => \'kyuukei_end\',
                        ], 
                    ]
                ];
            case 21:
                return 
                [
                    \'status_text\' => \'出勤中\',
                    \'buttons\' => [
                        [
                            \'button_text\' => \'退勤\',
                            \'action_name\' => \'taikin\',
                        ], 
                    ]
                ]; 
                
            case 99:
                return 
                [
                    \'status_text\' => \'退勤\',
                    \'buttons\' => [
                    ]
                ];
        }
    }
}
```

*   `PluginDashboardBase`を継承しており、ダッシュボードプラグインの基底機能を提供します。
*   `body()`メソッド:
    *   `dakoku`という名前のカスタムテーブルが存在するかを確認します。
    *   `getDakoku()`、`getStatus()`、`getStatusParams()`メソッドを呼び出して、現在の打刻状況とそれに応じた表示パラメータを取得します。
    *   `exment_dakoku::dakoku`というBladeテンプレートを呼び出し、打刻情報と表示パラメータを渡してダッシュボードに表示します。
*   `post()`メソッド:
    *   POSTリクエストで受け取った`action`パラメータ（`syukkin`, `kyuukei_start`, `kyuukei_end`, `taikin`）に応じて、打刻テーブルに現在時刻とステータスを保存します。
    *   打刻データがない場合は新規作成し、既存の場合は更新します。
    *   保存成功のメッセージを表示し、前のページに戻ります。
*   `getDakoku()`メソッド:
    *   `dakoku`カスタムテーブルから、現在のユーザーの今日の打刻データを取得します。午前4時を日付の区切りとして扱います。
*   `getStatus($dakoku)`メソッド:
    *   現在の打刻データに基づいて、打刻ステータス（未出勤、出勤中、休憩中、退勤）を数値で返します。
*   `getStatusParams($status)`メソッド:
    *   打刻ステータスに応じて、表示するテキストとボタンの情報を返します。

**このプラグインから学べること**:

*   Exmentにおけるダッシュボードプラグインの基本的な構造と`config.json`でのルーティング設定方法。
*   `PluginDashboardBase`を継承したクラスでのダッシュボードウィジェットの実装方法。
*   `CustomTable`モデルを使用してExmentのカスタムテーブルにデータを保存・取得する方法。
*   Bladeテンプレートを使用してUIを構築し、動的に表示内容を切り替える方法。
*   Laravelの`Carbon`ライブラリを使用した日付・時刻の操作。




### ドキュメントプラグインのサンプル: PluginDemoDocument

**概要**: `PluginDemoDocument`は、Exmentのカスタムデータ詳細画面から、Exmentの新着情報を取得し、Excelファイルとして出力するドキュメントプラグインです。これにより、Exmentの最新情報をオフラインで確認したり、レポートとして利用したりできます。

**`config.json`の要約**:

```json
{
    "plugin_name": "PluginDemoDocument",
    "uuid": "63d80590-4aad-11e9-b475-0800200c9a66",
    "plugin_view_name": "Plugin Demo Document",
    "description": "独自処理を実装してドキュメント出力を行うサンプルです。",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "document",
    "target_tables": "information",
    "filename": "Exment更新履歴_${ymdhms}",
    "label": "更新履歴出力",
    "icon": "fa-files-o",
    "button_class": "btn-success"
}
```

*   `plugin_type`: `document`として定義されており、ドキュメントプラグインであることを示します。
*   `target_tables`: `information`に設定されており、`information`テーブルのデータ詳細画面でこのプラグインが利用可能であることを示します。
*   `filename`: 出力されるファイル名が`Exment更新履歴_`にタイムスタンプが付加されたものになることを示します。
*   `label`: ボタンのラベルが「更新履歴出力」になることを示します。

**`Plugin.php`の要約**:

```php
<?php
namespace App\Plugins\PluginDemoDocument;

use Exceedone\Exment\Model\Define;
use Exceedone\Exment\Model\System;
use Exceedone\Exment\Services\Plugin\PluginDocumentBase;

class Plugin extends PluginDocumentBase
{
    /**
     * ドキュメントの変数置き換え実行前に呼び出される関数
     *
     * @param SpreadSheet $spreadsheet
     * @return void
     */
    protected function called($spreadsheet)
    {
        // ドキュメントの変数置き換え実行前に独自処理を実行したい場合はこちら
    }

    /**
     * ドキュメントの変数置き換え実行後に呼び出される関数
     *
     * @param SpreadSheet $spreadsheet
     * @return void
     */
    protected function saving($spreadsheet)
    {
        // １枚目のシートを選択
        $sheet = $spreadsheet->getSheet(0);
        // B1セルにサイト名を設定
        $sheet->setCellValue(\'B1\', System::site_name());

        // Exmentの新着情報を取得する
        $items = $this->getItems();

        foreach ($items as $idx => $item) {
            // 4行目～出力
            $row = $idx + 4;
            // 日付をA?セルに出力
            $date = \Carbon\Carbon::parse(array_get($item, \'date\'))->format(config(\'admin.date_format\'));
            $sheet->setCellValue("A$row", $date);
            // リンクをB?セルに出力
            $link = array_get($item, \'link\');
            $sheet->setCellValue("B$row", $link);
            // タイトルをC?セルに出力
            $title = array_get($item, \'title.rendered\');
            $sheet->setCellValue("C$row", $title);
        }
    }

    /**
     * Exmentの新着情報をAPIから取得する
     *
     * @return array exment news items array
     */
    protected function getItems()
    {
        $client = new \GuzzleHttp\Client();
        $response = $client->request(\'GET\', Define::EXMENT_NEWS_API_URL, [
            \'http_errors\' => false,
            \'query\' => $this->getQuery(),
            \'timeout\' => 3, // Response timeout
            \'connect_timeout\' => 3, // Connection timeout
        ]);

        $contents = $response->getBody()->getContents();
        if ($response->getStatusCode() != 200) {
            return [];
        }
        return json_decode_ex($contents, true);
    }

    /**
     * APIに渡すクエリ条件
     *
     * @return array query string array
     */
    protected function getQuery()
    {
        $request = request();

        // get querystring
        $query = [
            \'categories\' => 6,
            \'per_page\' => System::datalist_pager_count() ?? 5,
            \'page\' => $request->get(\'page\') ?? 1,
        ];

        return $query;
    }

    /**
    * (v3.4.3対応)画面にボタンを表示するかどうかの判定。デフォルトはtrue
    * 
    * @return bool true: 描写する false 描写しない
    */
    public function enableRender(){
        // カスタム列の値「view_flg」が「1」の場合にボタンを表示する
        return $this->custom_value->getValue(\'view_flg\')  == 1;
    }
}
```

*   `PluginDocumentBase`を継承しており、ドキュメントプラグインの基底機能を提供します。
*   `called($spreadsheet)`メソッド: ドキュメントの変数置き換え実行前に呼び出されます。このサンプルでは特に処理は記述されていません。
*   `saving($spreadsheet)`メソッド: ドキュメントの変数置き換え実行後に呼び出されます。このメソッド内でExcelシートにデータを書き込む処理が実装されています。
    *   `getItems()`メソッドでExmentの新着情報をAPIから取得します。
    *   取得した新着情報をExcelシートの指定されたセルに書き込みます。
*   `getItems()`メソッド: GuzzleHttpクライアントを使用して、`Define::EXMENT_NEWS_API_URL`からExmentの新着情報を取得します。
*   `getQuery()`メソッド: APIに渡すクエリ条件（カテゴリ、1ページあたりの表示数、ページ番号）を生成します。
*   `enableRender()`メソッド: (v3.4.3対応) ドキュメント出力ボタンを表示するかどうかの判定を行います。このサンプルでは、カスタム列`view_flg`の値が`1`の場合にボタンを表示します。

**このプラグインから学べること**:

*   Exmentにおけるドキュメントプラグインの基本的な構造と`config.json`での設定方法。
*   `PluginDocumentBase`を継承したクラスでのドキュメント生成ロジックの実装方法。
*   `called()`と`saving()`メソッドの役割と、Excelシートへのデータ書き込み方法。
*   外部API（Exmentの新着情報API）からデータを取得し、それをドキュメントに組み込む方法。
*   `enableRender()`メソッドによるボタン表示の制御方法。




### ドキュメントプラグインのサンプル: Docurain

**概要**: `Docurain`プラグインは、Exmentのデータ詳細画面からExcelテンプレートを使用してPDFファイルを生成する機能を提供します。これは、請求書、契約書、レポートなどの定型文書をExmentのデータに基づいて自動生成する際に非常に有用です。

**`config.json`の要約**:

```json
{
    "plugin_name": "Docurain",
    "uuid": "d680c0b9-6d42-4d76-b6a3-11f172902625",
    "plugin_view_name": "Docurain",
    "description": "ExcelファイルからPDFファイルを作成します。",
    "author": "Kajitori Co., Ltd.",
    "version": "1.0.1",
    "plugin_type": "trigger",
    "event_triggers": "form_menubutton_show",
    "label": "PDF出力",
    "icon": "fa-file-pdf-o"
}
```

*   `plugin_type`: `trigger`として定義されていますが、`form_menubutton_show`イベントでトリガーされるため、実質的にはドキュメント生成ボタンとして機能します。
*   `event_triggers`: `form_menubutton_show`に設定されており、フォームのメニューボタンが表示される際にこのプラグインがトリガーされることを示します。
*   `label`: ボタンのラベルが「PDF出力」になることを示します。

**`Plugin.php`の要約**:

```php
<?php
namespace App\Plugins\Docurain;

use Exceedone\Exment\Enums\FileType;
use Exceedone\Exment\Enums\ColumnType;
use Exceedone\Exment\Model\System;
use Exceedone\Exment\Services\Plugin\PluginTriggerBase;
use Exceedone\Exment\Model\File as ExmentFile;

class Plugin extends PluginTriggerBase
{
    protected $useCustomOption = true;
    protected const DOCURAIN_ERROR_MESSAGE = \'エラーが発生しました\';
    protected const DOCURAIN_API_URI = \'https://api.docurain.jp/api\';
    protected const DOCURAIN_ERROR_CODES = [
        // ... Docurain APIのエラーコード定義 ...
    ];
    
    /**
     * Plugin Trigger
     */
    public function execute()
    {
        $token = $this->plugin->getCustomOption(\'docurain_token\');

        if(!isset($token)){
            return [
                \'result\' => false,
                \'swal\' => static::DOCURAIN_ERROR_MESSAGE,
                \'swaltext\' => \'トークンがありません。プラグインの設定を行ってください。\',
            ];
        }

        list($filePath, $fileName, $outFileName) = $this->getTargetFile();
        if(!isset($filePath)){
            return [
                \'result\' => false,
                \'swal\' => static::DOCURAIN_ERROR_MESSAGE,
                \'swaltext\' => \'ファイル設定が行われていません。プラグインの設定を行ってください。\',
            ];
        }

        if(!\File::exists($filePath)){
            return [
                \'result\' => false,
                \'swal\' => static::DOCURAIN_ERROR_MESSAGE,
                \'swaltext\' => "ファイル「{\$fileName}」がありませんでした。",
            ];
        }

        $body = fopen($filePath, \'r\');

        $postValue = $this->postValue();

        $client = new \GuzzleHttp\Client();

        $response = $client->request(\'POST\', url_join(static::DOCURAIN_API_URI, \'instant\', \'pdf\'), [
            \'http_errors\' => false,
            \'headers\' => [
                \'Authorization\' => "token {
                    \"href\" => $value->getUrl(),
                    \"target\" => \"_top\",
                    ($key ? \'icon_right\' : \'icon\') => $key ? \'fa-arrow-right\' : \'fa-arrow-left\',
                    \'label\' => $value->getLabel(),
                ];
            }
        }

        // button.blade.phpでボタンを描写する
        return $this->pluginView(\'buttons\', [\'buttons\' => $buttons]);
    }

    /**
     * 次もしくは前のデータを取得する
     *
     * @param boolean $isNext true: 次のデータ、false: 前のデータ
     * @return CustomValue
     */
    protected function getNextOrPrevId(bool $isNext){
        $query = $this->custom_table->getValueModel()->query();   
        $query->where(\'id\', ($isNext ? \'>\' : \'<\'), $this->custom_value->id);

        $query->orderBy(\'id\', ($isNext ? \'asc\' : \'desc\'));

        return $query->first();
    }
}
```

*   `PluginTriggerBase`を継承していますが、`execute()`メソッド内でDocurain APIを呼び出してPDFを生成するロジックが実装されています。
*   `$useCustomOption = true;`: カスタム設定オプションを有効にします。
*   `execute()`メソッド:
    *   プラグイン設定でDocurainトークンが設定されているか、テンプレートファイルが指定されているかを確認します。
    *   GuzzleHttpクライアントを使用して、Docurain APIにExcelテンプレートファイルとExmentのデータ（`postValue()`で生成）を送信し、PDFを生成します。
    *   生成されたPDFファイルをExmentのファイルストレージに保存し、カスタム値に紐付けます。
    *   APIからのエラーレスポンスを処理し、ユーザーにエラーメッセージを表示します。
*   `setCustomOptionForm(&$form)`メソッド:
    *   プラグインの設定画面に、Docurainトークン入力フィールドと、テーブル名と帳票ファイル名の一覧を入力するテキストエリアを追加します。
*   `getButtonLabel()`メソッド:
    *   プラグイン設定で指定されたテーブル名とファイル名に基づいて、ボタンのラベルを動的に取得します。
*   `getTargetFile()`メソッド:
    *   プラグイン設定で指定されたテーブル名とファイル名に基づいて、使用するExcelテンプレートファイルのパスと出力ファイル名を取得します。
*   `postValue()`メソッド:
    *   現在のカスタム値のデータ、親テーブルのデータ、関連する他のテーブルのデータ、子テーブルのデータ、システムパラメータなど、Docurain APIに送信するデータを整形して返します。
*   `replaceBlank()`メソッド:
    *   Docurainでテンプレート設定値がそのまま表示されるのを防ぐため、`null`値を空文字に変換します。
*   `getErrorMessage()`メソッド:
    *   Docurain APIからのエラーレスポンスを解析し、ユーザーに分かりやすいエラーメッセージを生成します。

**このプラグインから学べること**:

*   Exmentにおけるトリガープラグイン（特にボタンとして機能するタイプ）の応用例。
*   `PluginTriggerBase`を継承したクラスで外部API（Docurain）と連携し、ドキュメントを生成する方法。
*   `$useCustomOption`と`setCustomOptionForm()`メソッドを使用して、プラグインにカスタム設定画面を追加する方法。
*   `ExmentFile::storeAs()`を使用して、生成されたファイルをExmentのファイルストレージに保存する方法。
*   `CustomTable`、`CustomValue`、`System`モデルなど、Exmentの内部データ構造にアクセスして、必要な情報を取得し、外部サービスに渡す方法。
*   GuzzleHttpクライアントを使用した外部APIとの連携。
*   複雑なデータ構造を外部APIの要件に合わせて整形する方法。




### イベントプラグインのサンプル: PluginSyncCity

**概要**: `PluginSyncCity`は、Exmentの「city」テーブルのデータが保存された際に、外部データベースの「city」テーブルと同期するイベントプラグインです。これにより、Exmentと外部システム間で都市データをリアルタイムに連携させることができます。

**`config.json`の要約**:

```json
{
    "plugin_name": "PluginSyncCity",
    "uuid": "539a04f5-aee2-cb63-326e-ea1e1a922d6f",
    "plugin_view_name": "外部データ連携",
    "description": "都市データの情報を外部データベースと連携します。",
    "author": "Kajitori",
    "version": "1.0.0",
    "target_tables": "city",
    "plugin_type": "event",
    "event_triggers": "saved"
}
```

*   `plugin_type`: `event`として定義されており、イベントプラグインであることを示します。
*   `target_tables`: `city`に設定されており、`city`テーブルのデータが対象であることを示します。
*   `event_triggers`: `saved`に設定されており、データが保存されたイベントでこのプラグインがトリガーされることを示します。

**`Plugin.php`の要約**:

```php
<?php
namespace App\Plugins\PluginSyncCity;

use Exceedone\Exment\Model\Define;
use Exceedone\Exment\Services\Plugin\PluginEventBase;
use Illuminate\Support\Str;

class Plugin extends PluginEventBase
{
    protected $useCustomOption = true;

    /**
     * Plugin Trigger
     */
    public function execute()
    {
        // プラグイン設定から外部データベース接続情報を取得し、動的に設定
        config([\"database.connections.plugin_connection\" => [
            \"driver\"    => $this->plugin->getCustomOption(\'custom_driver\', \'mysql\'),
            \"host\"      => $this->plugin->getCustomOption(\'custom_host\', \'127.0.0.1\'),
            \"port\"  => $this->plugin->getCustomOption(\'custom_port\', \'3306\'),
            \"database\"  => $this->plugin->getCustomOption(\'custom_database\', \'test\'),
            \"username\"  => $this->plugin->getCustomOption(\'custom_user\', \'root\'),
            \"password\"  => $this->plugin->getCustomOption(\'custom_password\', \'password\'),
            \"charset\"   => \'utf8\',
            \"collation\" => \'utf8_unicode_ci\'
        ]]);

        $country_code = \'JPN\';
        $name_en = Str::ucfirst(Str::lower($this->custom_value->getValue(\'name_en\')));
        $population = $this->custom_value->getValue(\'population\');
        $prefectures = Str::ucfirst(Str::lower($this->custom_value->getValue(\'prefectures\')));

        // 外部データベースから都市データを検索
        $city = \DB::connection(\'plugin_connection\')
            ->table(\'city\')
            ->where(\'CountryCode\', $country_code)
            ->where(\'Name\', $name_en)
            ->first();

        if (isset($city)) {
            if ($this->isDelete) { // 削除イベントの場合
                $result = \DB::connection(\'plugin_connection\')
                    ->table(\'city\')
                    ->where(\'ID\', $city->ID)
                    ->delete();
            } else { // 更新イベントの場合
                $result = \DB::connection(\'plugin_connection\')
                    ->table(\'city\')
                    ->where(\'ID\', $city->ID)
                    ->update([
                        \"Population\" => $population?? $city->Population,
                        \"District\" => $prefectures?? $city->District
                    ]);
            }
        } else { // 新規作成イベントの場合
            $result = \DB::connection(\'plugin_connection\')
                ->table(\'city\')
                ->insert([
                    \"Name\" => $name_en,
                    \"CountryCode\" => $country_code,
                    \"Population\" => $population,
                    \"District\" => $prefectures
                ]);
        }

        return $result;
    }

    /**
     * カスタムオプション（外部データベース接続先）
     *
     * @param $form
     * @return void
     */
    public function setCustomOptionForm(&$form)
    {
        $form->exmheader(\'外部データベースの情報\');
        $form->select(\'custom_driver\', \'データベースの種類\')
            ->options(Define::DATABASE_TYPE)
            ->default(\'mysql\');
        $form->text(\'custom_host\', \'ホスト名\')
            ->default(\'127.0.0.1\');
        $form->text(\'custom_port\', \'ポート番号\')
            ->default(\'3306\');
        $form->text(\'custom_database\', \'データベース名\');
        $form->text(\'custom_user\', \'ユーザー名\')
            ->default(\'root\');
        $form->password(\'custom_password\', \'パスワード\');
    }
}
```

*   `PluginEventBase`を継承しており、イベントプラグインの基底機能を提供します。
*   `$useCustomOption = true;`: カスタム設定オプションを有効にします。
*   `execute()`メソッド:
    *   プラグイン設定で定義されたカスタムオプション（外部データベースの接続情報）を使用して、動的にデータベース接続を設定します。
    *   Exmentの`city`テーブルから取得したデータ（`name_en`, `population`, `prefectures`）を基に、外部データベースの`city`テーブルを検索します。
    *   データが存在する場合は、Exmentでの操作（作成、更新、削除）に応じて外部データベースのレコードを更新または削除します。
    *   データが存在しない場合は、外部データベースに新しいレコードを挿入します。
*   `setCustomOptionForm(&$form)`メソッド:
    *   プラグインの設定画面に、外部データベース接続のためのフォーム要素（データベースの種類、ホスト名、ポート番号、データベース名、ユーザー名、パスワード）を追加します。

**このプラグインから学べること**:

*   Exmentにおけるイベントプラグインの基本的な構造と`config.json`でのイベントトリガー、対象テーブルの設定方法。
*   `PluginEventBase`を継承したクラスでのイベント処理の実装方法。
*   `$useCustomOption`と`setCustomOptionForm()`メソッドを使用して、プラグインにカスタム設定画面を追加し、外部データベース接続情報を動的に設定する方法。
*   `DB::connection()`を使用して、Exmentから外部データベースに接続し、データを同期する方法。
*   `isDelete`プロパティを使用して、イベントが削除操作によるものかを判断する方法。




### エクスポートプラグインのサンプル: ExportTestCsv

**概要**: `ExportTestCsv`は、ExmentのデータをCSV形式でエクスポートするプラグインです。特定のテーブルのデータを独自のCSV形式で出力する機能を提供します。

**`config.json`の要約**:

```json
{
    "plugin_name": "ExportTestCsv",
    "uuid": "1e7881d0-324f-11e9-b56e-0800200c9a33",
    "plugin_view_name": "エクスポート(CSV)",
    "description": "エクスポートのCSVテストです。",
    "author": "Kajitori",
    "version": "0.0.1",
    "plugin_type": "export",
    "target_tables": "information",
    "label": "お知らせ情報出力",
    "icon": "fa-question",
    "export_description": "お知らせ情報を、独自のcsv形式で一覧出力します。"
}
```

*   `plugin_type`: `export`として定義されており、エクスポートプラグインであることを示します。
*   `target_tables`: `information`に設定されており、`information`テーブルのデータが対象であることを示します。
*   `label`: ボタンのラベルが「お知らせ情報出力」になることを示します。
*   `export_description`: エクスポート機能の説明が記述されています。

**`Plugin.php`の要約**:

```php
<?php
namespace App\Plugins\ExportTestCsv;

use Exceedone\Exment\Services\Plugin\PluginExportBase;

class Plugin extends PluginExportBase
{
    /**
     * execute
     */
    public function execute() 
    {
        // ※メソッド「$this->getTmpFullPath()」で、一時tmpファイルを取得する
        // ※実行後、一時tmpファイルは自動的に削除されます。
        $tmp = $this->getTmpFullPath();

        // csvファイルを開く
        $fp = fopen($tmp, \'w\');

        // すべてのシステム列・カスタム列でデータ一覧取得（配列）
        $data = $this->getData();

        // ビュー形式でデータ一覧取得（配列）
        // $data = $this->getViewData();

        // CustomValueのCollectionでデータ一覧取得
        // $data = $this->getRecords();


        foreach ($data as $fields) {
            fputcsv($fp, $fields);
        }

        fclose($fp);

        // $tmpのstring文字列を返却する
        return $tmp;
    }

    /**
     * Get download file name.
     * ファイル名を取得する
     *
     * @return string
     */
    public function getFileName() : string {
        return \"test.csv\";
    }
}
```

*   `PluginExportBase`を継承しており、エクスポートプラグインの基底機能を提供します。
*   `execute()`メソッド:
    *   一時ファイルを作成し、そのパスを取得します。
    *   `getData()`メソッド（または`getViewData()`、`getRecords()`）でExmentからデータを取得します。
    *   取得したデータをCSV形式で一時ファイルに書き込みます。
    *   一時ファイルのパスを返します。このファイルはエクスポート後に自動的に削除されます。
*   `getFileName()`メソッド: ダウンロードされるCSVファイルの名前を返します。この例では`test.csv`です。

**このプラグインから学べること**:

*   Exmentにおけるエクスポートプラグインの基本的な構造と`config.json`での設定方法。
*   `PluginExportBase`を継承したクラスでのエクスポート処理の実装方法。
*   `getData()`、`getViewData()`、`getRecords()`といったメソッドを使用してExmentからデータを取得する方法。
*   一時ファイルを作成し、そこにデータを書き込む方法。
*   エクスポートされるファイル名を指定する方法。




### インポートプラグインのサンプル: PluginImportContract

**概要**: `PluginImportContract`は、Excelファイルから契約データを読み込み、Exmentの「契約」テーブルと「契約明細」テーブルにインポートするプラグインです。独自のロジックで複雑なデータ構造を持つファイルを処理し、Exmentに登録する例を示しています。

**`config.json`の要約**:

```json
{
    "plugin_name": "PluginImportContract",
    "uuid": "63d80590-4aad-11e9-b475-0800200c9a66",
    "plugin_view_name": "独自ロジックによるインポート",
    "description": "契約データを独自ロジックでインポートします。",
    "author": "Kajitori",
    "version": "1.0.0",
    "plugin_type": "import",
    "target_tables": "contract",
    "label": "契約インポート",
    "icon": "fa-files-o"
}
```

*   `plugin_type`: `import`として定義されており、インポートプラグインであることを示します。
*   `target_tables`: `contract`に設定されており、`contract`テーブルがインポートの対象であることを示します。
*   `label`: ボタンのラベルが「契約インポート」になることを示します。

**`Plugin.php`の要約**:

```php
<?php
namespace App\Plugins\PluginImportContract;

use Exceedone\Exment\Services\Plugin\PluginImportBase;
use PhpOffice\PhpSpreadsheet\IOFactory;

class Plugin extends PluginImportBase{
    /**
     * execute
     */
    public function execute() {
        $path = $this->file->getRealPath(); // アップロードされたファイルのパスを取得

        $reader = $this->createReader();
        $spreadsheet = $reader->load($path); // Excelファイルを読み込む

        // Sheet1のB4セルの内容で顧客マスタを読み込みます
        $sheet = $spreadsheet->getSheetByName(\'Sheet1\');
        $client_name = getCellValue(\'B4\', $sheet, true);
        $client = getModelName(\'client\')::where(\'value->client_name\', $client_name)->first();

        // Sheet1のヘッダ部分に記載された情報で契約データを編集します
        // statusには固定値:1を設定します
        $contract = [
            \"value->contract_code\" => getCellValue(\'B3\', $sheet, true),
            \"value->client\" => $client->id,
            \"value->status\" => \'1\',
            \"value->contract_date\" => getCellValue(\'D4\', $sheet, true),
        ];
        // 契約テーブルにレコードを追加します
        $record = getModelName(\'contract\')::create($contract);

        // Sheet1の7行目～15行目に記載された明細情報を元に契約明細データを出力します
        for ($i = 7; $i <= 15; $i++) {
            // A列から製品バージョンコードを取得します
            $product_version_code = getCellValue(\"A$i\", $sheet, true);
            // 製品バージョンコードが設定されていない時は次の行にスキップします
            if (!isset($product_version_code)) break;
            // 製品バージョンコードで、製品バージョンテーブルを読み込みます
            $product_version = getModelName(\'product_version\')
                ::where(\'value->product_version_code\', $product_version_code)->first();
            // 製品バージョンテーブルが取得できなかった場合は次の行にスキップします
            if (!isset($product_version)) continue;
            // 明細行と製品バージョンテーブルから契約明細データを編集します
            $contract_detail = [
                \"parent_id\" => $record->id,
                \"parent_type\" => \"contract\",
                \"value->product_version_id\" => $product_version->id,
                \"value->fixed_price\" => getCellValue(\"B$i\", $sheet, true),
                \"value->num\" => getCellValue(\"C$i\", $sheet, true),
                \"value->zeinuki_price\" => getCellValue(\"D$i\", $sheet, true),
            ];
            // 契約明細テーブルにレコードを追加します
            getModelName(\'contract_detail\')::create($contract_detail);
        }

        return true;
    }
    
    protected function createReader()
    {
        return IOFactory::createReader(\'Xlsx\');
    }
    
}
```

*   `PluginImportBase`を継承しており、インポートプラグインの基底機能を提供します。
*   `execute()`メソッド:
    *   アップロードされたExcelファイルを`PhpOffice\PhpSpreadsheet`ライブラリを使用して読み込みます。
    *   `Sheet1`の特定のセルから顧客名、契約コード、契約日などの情報を取得し、Exmentの「顧客」テーブルから顧客情報を検索します。
    *   取得した情報と固定値（`status: 1`）を使用して、Exmentの「契約」テーブルに新しいレコードを作成します。
    *   `Sheet1`の指定された行範囲から明細情報をループで読み込みます。
    *   各明細行から製品バージョンコードを取得し、Exmentの「製品バージョン」テーブルから製品情報を検索します。
    *   取得した情報と親契約のIDを使用して、Exmentの「契約明細」テーブルに新しいレコードを作成します。
*   `createReader()`メソッド: Excelファイルを読み込むための`PhpOffice\PhpSpreadsheet`のリーダーを返します。

**このプラグインから学べること**:

*   Exmentにおけるインポートプラグインの基本的な構造と`config.json`での設定方法。
*   `PluginImportBase`を継承したクラスでのインポート処理の実装方法。
*   `PhpOffice\PhpSpreadsheet`ライブラリを使用してExcelファイルを読み込み、特定のセルからデータを取得する方法。
*   Exmentの既存テーブル（顧客、製品バージョン）から関連データを検索し、新規レコード（契約、契約明細）を作成する方法。
*   複雑なExcelシートの構造から、複数のExmentテーブルにデータをインポートするロジックの実装例。




### 複数プラグインタイプ対応のサンプル: BatchButton

**概要**: `BatchButton`は、Exmentのバッチプラグインとボタンプラグインを1つのプラグインとして同時に実装するサンプルです。共通クラスを呼び出すことで、両方のプラグインタイプで共通のロジックを再利用できます。

**`config.json`の要約**:

```json
{
    "uuid": "697b7391-4f14-42c9-2f18-be57a56b0f66",
    "plugin_name": "BatchButton",
    "plugin_view_name": "バッチ・ボタンの同時実装",
    "description": "バッチ・ボタンのプラグインを、同時に実装します。",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "batch,button",
    "event_triggers": "form_menubutton_show",
    "target_tables": "information"
}
```

*   `plugin_type`: `batch,button`とカンマ区切りで複数指定されており、このプラグインがバッチとボタンの両方のタイプをサポートすることを示します。
*   `event_triggers`: `form_menubutton_show`に設定されており、ボタン機能がフォームのメニューボタン表示時にトリガーされることを示します。
*   `target_tables`: `information`に設定されており、ボタン機能が`information`テーブルにのみ適用されることを示します。

**`PluginBatch.php`の要約**:

```php
<?php

namespace App\Plugins\BatchButton;

use Exceedone\Exment\Services\Plugin\PluginBatchBase;

class PluginBatch extends PluginBatchBase
{
    /**
     */
    public function execute()
    {
        Common::log($this->plugin);
    }
}
```

*   `PluginBatchBase`を継承しており、バッチプラグインとしての機能を提供します。
*   `execute()`メソッド内で、共通クラス`Common::log()`を呼び出しています。

**`PluginButton.php`の要約**:

```php
<?php

namespace App\Plugins\BatchButton;

use Exceedone\Exment\Services\Plugin\PluginButtonBase;

class PluginButton extends PluginButtonBase
{
    /**
     * @param [type] $form
     * @return void
     */
    public function execute()
    {
        Common::log($this->plugin);
    }
}
```

*   `PluginButtonBase`を継承しており、ボタンプラグインとしての機能を提供します。
*   `execute()`メソッド内で、共通クラス`Common::log()`を呼び出しています。

**`Common.php`の要約**:

```php
<?php

namespace App\Plugins\BatchButton;

class Common
{
    public static function log($plugin)
    {
        \Log::debug($plugin->getCustomOption(\'text\') ?? \'Executed!\');
    }
}
```

*   `Common`クラスは、`PluginBatch`と`PluginButton`の両方から呼び出される共通のロジックを提供します。
*   `log()`静的メソッドは、プラグインのカスタムオプションから`text`を取得し、デバッグログに出力します。

**`PluginSetting.php`の要約**:

```php
<?php

namespace App\Plugins\BatchButton;

use Exceedone\Exment\Services\Plugin\PluginSettingBase;

class PluginSetting extends PluginSettingBase
{
    protected $useCustomOption = true;

    /**
     * @param [type] $form
     * @return void
     */
    public function setCustomOptionForm(&$form)
    {
        $form->text(\'text\', \'ログ出力文字列\')
            ->help(\'ログに出力する文字列を設定します。\')
            ->default(\'Executed!\');
    }
}
```

*   `PluginSettingBase`を継承しており、プラグインのカスタム設定画面を提供します。
*   `setCustomOptionForm()`メソッドで、ログに出力する文字列を設定するためのテキストフィールドを追加します。

**このプラグインから学べること**:

*   1つのプラグインで複数の`plugin_type`をサポートする方法。
*   各プラグインタイプに対応するPHPファイルを`PluginBatch.php`、`PluginButton.php`のように命名する方法。
*   共通のロジックを`Common.php`のような独立したクラスに切り出して再利用する方法。
*   `PluginSetting.php`を使用して、複数のプラグインタイプに対応するプラグインのカスタム設定を管理する方法。




### ページプラグインのサンプル: SystemTableColumnList

**概要**: `SystemTableColumnList`は、Exmentの内部パラメータも含めたテーブルと列の一覧を表示するページプラグインです。Exmentのデータ構造を理解し、デバッグや開発に役立つ情報を提供します。

**`config.json`の要約**:

```json
{
    "plugin_name": "SystemTableColumnList",
    "plugin_view_name" : "システム向け テーブル・列一覧表示",
    "explain": "内部パラメータも含めた、Exmentのテーブル・列の一覧を表示します。",
    "uuid":  "40a55717-6aed-bd06-93d9-749667e26ce6",
    "author":  "",
    "version": "1.0.0",
    "plugin_type": "page",
    "route": [
        {
            "uri": "",
            "method": [
                "get"
            ],
            "function": "index"
        }
    ],
    "templates": true
}
```

*   `plugin_type`: `page`として定義されており、ページプラグインであることを示します。
*   `route`: ベースURI（空文字列）へのGETリクエストで`index`メソッドが実行されるように設定されています。
*   `templates`: `true`に設定されており、Bladeテンプレートを使用することを示します。

**`Plugin.php`の要約**:

```php
<?php

namespace App\Plugins\SystemTableColumnList;

use Encore\Admin\Widgets\Box;
use Encore\Admin\Widgets\Table;
use Exceedone\Exment\Services\Plugin\PluginPageBase;
use Exceedone\Exment\Model\CustomTable;
use Encore\Admin\Widgets\Form as WidgetForm;
use Exceedone\Exment\Enums\ColumnType;

class Plugin extends PluginPageBase
{
    /**
     * Display a listing of the resource.
     *
     * @return Content|\Illuminate\Http\Response
     */

    public function index()
    {
        // テーブルの一覧を取得
        $tables = CustomTable::filterList();

        // テーブルを指定していた場合、そのテーブルを絞り込み
        $table = CustomTable::getEloquent(request()->get(\'table\'));
        
        $html = \'\';

        $form = new WidgetForm();
        $form->disableReset();
        $form->disableSubmit();

        $form->description(\'内部のシステム名も含めた、Exmentのテーブル・列の一覧を表示します。テーブルを選択してください。\');

        $form->select(\'table\', \'該当テーブル\')->options($tables->pluck(\'table_view_name\', \'table_name\'))
            ->setElementClass(\'system_table_column_list_select_table\')
            ->default($table ? $table->table_name : null);

        ///// テーブルが存在する場合のみ、以下を実施
        if ($table) {
            $form->display(exmtrans(\'custom_table.table_name\'))->default($table->table_name);
            $form->display(exmtrans(\'custom_table.table_view_name\'))->default($table->table_view_name);
            $form->display(\'データベース テーブル名\')->default(getDBTableName($table));
            $form->display(exmtrans(\'common.suuid\'))->default($table->suuid);

            // 列一覧を表示
            $form->exmheader(\'列一覧\')->hr();
        }

        // リロード用のURL
        $url = $this->plugin->getFullUrl();
        $form->hidden(\'system_table_column_list_root_url\')->default($url);

        $html .= $form->render();
        
        if ($table) {
            $headers = [
                exmtrans(\'custom_column.column_name\'),
                exmtrans(\'custom_column.column_view_name\'),
                exmtrans(\'custom_column.column_type\'),
                exmtrans(\'custom_column.options.index_enabled\'),
                \'データベース 列名\',
                exmtrans(\'common.suuid\'),
            ];
            $bodies = [];
            foreach($table->custom_columns_cache as $custom_column){
                $body = [];
                $body[] = esc_html($custom_column->column_name);
                $body[] = esc_html($custom_column->column_view_name);
                $body[] = array_get(ColumnType::transArray(\"custom_column.column_type_options\"), $custom_column->column_type);
                $body[] = \Exment::getTrueMark($custom_column->index_enabled);
                $body[] = esc_html($custom_column->getQueryKey());
                $body[] = esc_html($custom_column->suuid);

                $bodies[] = $body;
            }

            $html .= (new Table($headers, $bodies))->render();
        }

        $box = new Box(\'システム用 テーブル・列一覧\', $html);

        return $box;
    }
}
```

*   `PluginPageBase`を継承しており、ページプラグインの基底機能を提供します。
*   `index()`メソッド:
    *   `CustomTable::filterList()`でExment内のすべてのテーブルの一覧を取得します。
    *   リクエストパラメータで特定のテーブルが指定されている場合、そのテーブルに絞り込みます。
    *   Laravel Adminの`WidgetForm`を使用して、テーブル選択用のドロップダウンと、選択されたテーブルの詳細情報を表示するフォームを構築します。
    *   選択されたテーブルが存在する場合、そのテーブルのすべての列（カスタム列）の詳細情報（列名、表示名、タイプ、インデックス有効化、データベース列名、SUUIDなど）を`Table`ウィジェットを使用して表示します。
    *   最終的に、これらのフォームとテーブルを`Box`ウィジェットにまとめて返します。

**このプラグインから学べること**:

*   Exmentにおけるページプラグインの基本的な構造と`config.json`でのルーティング設定方法。
*   `PluginPageBase`を継承したクラスでのカスタムページの構築方法。
*   `CustomTable`モデルと`CustomColumn`モデルを使用してExmentの内部データ構造（テーブルと列のメタデータ）にアクセスする方法。
*   Laravel Adminの`WidgetForm`と`Table`ウィジェットを使用して、動的なフォームとデータテーブルをページに表示する方法。
*   Exmentのシステム情報を取得し、開発やデバッグに役立つツールを作成する方法。




### ページプラグインのサンプル: YasumiPage

**概要**: `YasumiPage`は、指定された年の日本の祝日を一覧表示するページプラグインです。`azuyalabs/yasumi`ライブラリを利用して祝日情報を取得し、ユーザーフレンドリーな形式で表示します。

**`config.json`の要約**:

```json
{
    "plugin_name": "YasumiPage",
    "plugin_view_name" : "休みページ",
    "explain": "表示年の祝日をページに表示します。",
    "uuid":  "616798f1-687d-9ce0-5e55-12f09f7bbf9a",
    "author":  "",
    "version": "1.0.0",
    "plugin_type": "page",
    "route": [
        {
            "uri": "",
            "method": [
                "get"
            ],
            "function": "index"
        },
        {
            "uri": "list",
            "method": [
                "get"
            ],
            "function": "list"
        }
    ],
    "requirement": [
        {
            "class": "\\Yasumi\\Yasumi",
            "composer": "azuyalabs/yasumi=^2.3"
        }
    ]
}
```

*   `plugin_type`: `page`として定義されており、ページプラグインであることを示します。
*   `route`: ベースURI（空文字列）へのGETリクエストで`index`メソッドが、`list`URIへのGETリクエストで`list`メソッドが実行されるように設定されています。
*   `requirement`: `azuyalabs/yasumi`というComposerパッケージが必要であることを示しています。

**`Plugin.php`の要約**:

```php
<?php

namespace App\Plugins\YasumiPage;

use Exceedone\Exment\Services\Plugin\PluginPageBase;
use Encore\Admin\Widgets\Box;
use Encore\Admin\Widgets\Table;

class Plugin extends PluginPageBase
{
    /**
     * Index
     *
     * @return void
     */
    public function index()
    {
        return $this->getIndexBox();
    }

    public function list()
    {
        $result = [];
        $weeks = [\"日\", \"月\", \"火\", \"水\", \"木\", \"金\", \"土\"];

        $years = range(request()->get(\'year_from\'), request()->get(\'year_to\'));

        foreach($years as $year){
            $holidays = \Yasumi\Yasumi::create(\'Japan\', $year, \'ja_JP\');
            foreach($holidays->getHolidays() as $holiday){
                $result[] = [
                    $holiday->format(\'Y/m/d\') . \'(\' . $weeks[$holiday->format(\'w\')] . \')\',
                    $holiday->getName(),
                ];
            }
        }
        
        $table = new Table([
            \"日付\",
            \"祝日名\",
        ], $result);

        $html = $this->getIndexBox()->render();
        $html .= (new Box(\"祝日検索結果\", $table))->render();

        return $html;
    }

    
    protected function getIndexBox(){
        // Yasumiのチェック
        $hasLibrary = class_exists(\\Yasumi\\Yasumi::class);
        $currentYear = \Carbon\Carbon::now()->year;

        return new Box(\"祝日一覧取得\", view(\'exment_yasumi_page::index\', [
            \\'action\\' => $this->plugin->getRouteUri(\'list\'),
            \\'hasLibrary\\' => $hasLibrary,
            \\'years\\' => range($currentYear - 10, $currentYear + 2),
            \\'selectYearFrom\\' => request()->get(\'year_from\', $currentYear - 1),
            \\'selectYearTo\\' => request()->get(\'year_to\', $currentYear + 1),
        ]));
    }
    
}
```

*   `PluginPageBase`を継承しており、ページプラグインの基底機能を提供します。
*   `index()`メソッド: 祝日検索フォームを表示するための`getIndexBox()`メソッドを呼び出します。
*   `list()`メソッド:
    *   リクエストパラメータから取得した開始年と終了年の範囲でループし、`Yasumi\Yasumi::create('Japan', $year, 'ja_JP')`を使用して各年の日本の祝日を取得します。
    *   取得した祝日情報を整形し、Laravel Adminの`Table`ウィジェットで表示します。
    *   検索フォームと祝日検索結果をまとめて表示します。
*   `getIndexBox()`メソッド: 祝日検索フォームを含む`Box`ウィジェットを返します。このフォームは、祝日検索の対象となる年を選択するためのものです。

**このプラグインから学べること**:

*   Exmentにおけるページプラグインの基本的な構造と`config.json`でのルーティング、外部ライブラリの要件定義方法。
*   `PluginPageBase`を継承したクラスでのカスタムページの構築方法。
*   `azuyalabs/yasumi`のような外部ライブラリをExmentプラグイン内で利用する方法。
*   Laravel Adminの`Box`と`Table`ウィジェットを使用して、動的なコンテンツをページに表示する方法。
*   ユーザーからの入力（年範囲）に基づいてデータを取得し、表示する方法。




### ページプラグインのサンプル: YouTubeSearch

**概要**: `YouTubeSearch`は、YouTubeの動画を検索し、その再生数などの統計データをExmentに保存するページプラグインです。YouTube Data APIを利用して動画情報を取得し、カスタムテーブルに保存する例を示しています。

**`config.json`の要約**:

```json
{
    "plugin_name": "YouTubeSearch",
    "plugin_view_name" : "YouTube検索",
    "explain": "YouTubeを検索し、再生数などをデータ保存します。",
    "uuid":  "c6d8daa0-c255-11e9-bb97-0800200c9a66",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "page",
    "route": [
        {
            "uri": "",
            "method": [
                "get"
            ],
            "function": "index"
        },
        {
            "uri": "list",
            "method": [
                "get"
            ],
            "function": "list"
        },
        {
            "uri": "save",
            "method": [
                "post"
            ],
            "function": "save"
        }
    ]
}
```

*   `plugin_type`: `page`として定義されており、ページプラグインであることを示します。
*   `route`: `index`、`list`、`save`の3つのルートが定義されており、それぞれ検索フォームの表示、検索結果の表示、データの保存を担当します。

**`Plugin.php`の要約**:

```php
<?php

namespace App\Plugins\YouTubeSearch;

use Encore\Admin\Form;
use Encore\Admin\Widgets\Box;
use Exceedone\Exment\Model\CustomTable;
use Exceedone\Exment\Services\Plugin\PluginPageBase;
use GuzzleHttp\Client;


class Plugin extends PluginPageBase
{
    protected $useCustomOption = true;

    /**
     * Index
     *
     * @return void
     */
    public function index()
    {
        return $this->getIndexBox();
    }

    /**
     * 一覧表示
     *
     * @return void
     */
    public function list()
    {
        $html = $this->getIndexBox()->render();

        // 文字列検索
        $client = new Client([
            \"base_uri\" => \"https://www.googleapis.com/youtube/v3/\",
        ]);

        $method = \"GET\";
        $uri = \"search?part=id&type=video&maxResults=20&key=\" . $this->plugin->getCustomOption(\'access_key\') 
            . \"&q=\" . urlencode(request()->get(\'youtube_search_query\')); //検索
        $options = [];
        $response = $client->request($method, $uri, $options);

        $list = json_decode($response->getBody()->getContents(), true);
        $ids = collect(array_get($list, \'items\', []))->map(function($l){
            return array_get($l, \'id.videoId\');
        })->toArray();


        // idより詳細を検索
        $client = new Client([
            \"base_uri\" => \"https://www.googleapis.com/youtube/v3/\",
        ]);

        $method = \"GET\";
        $uri = \"videos?part=id,snippet,statistics&key=\" . $this->plugin->getCustomOption(\'access_key\') 
            . \"&id=\" . implode(\\'\\', $ids); //検索
        $options = [];
        $response = $client->request($method, $uri, $options);

        $list = json_decode($response->getBody()->getContents(), true);
        
        $html .= new Box(\"YouTube検索結果\", view(\'exment_you_tube_search::list\', [
            \'items\' => array_get($list, \'items\', []),
            \'item_action\' => $this->plugin->getRouteUri(\'save\'),
        ])->render());

        return $html;
    }

    public function save(){
        $request = request();
        $model = CustomTable::getEloquent(\'youtube\')->getValueModel();
        $model->setValue(\'youtubeId\', $request->get(\'youtubeId\'));
        $model->setValue(\'description\', $request->get(\'description\'));
        $model->setValue(\'viewCount\', $request->get(\'viewCount\'));
        $model->setValue(\'likeCount\', $request->get(\'likeCount\'));
        $model->setValue(\'dislikeCount\', $request->get(\'dislikeCount\'));
        $model->setValue(\'url\', $request->get(\'url\'));
        $model->setValue(\'title\', $request->get(\'title\'));
        $model->setValue(\'publishedAt\', $request->get(\'publishedAt\'));
        $model->save();

        admin_toastr(trans(\'admin.save_succeeded\'));
        return redirect()->back()->withInput();
    }

    protected function getIndexBox(){
        // YouTube アクセスキーのチェック
        $hasKey = !is_null($this->plugin->getCustomOption(\'access_key\'));

        return new Box(\"YouTube検索\", view(\'exment_you_tube_search::index\', [
            \'action\' => $this->plugin->getRouteUri(\'list\'),
            \'youtube_search_query\' => request()->get(\'youtube_search_query\'),
            \'hasKey\' => $hasKey,
        ]));
    }
    
    /**
     * Set Custom Option Form. Using laravel-admin form option
     * https://laravel-admin.org/docs/#/en/model-form-fields
     *
     * @param [type] $form
     * @return void
     */
    public function setCustomOptionForm(&$form)
    {
        $form->text(\'access_key\', \'アクセスキー\')
            ->help(\'YouTubeのアクセスキーを入力してください。\');
    }
}
```

*   `PluginPageBase`を継承しており、ページプラグインの基底機能を提供します。
*   `$useCustomOption = true;`: カスタム設定オプションを有効にします。
*   `index()`メソッド: YouTube検索フォームを表示するための`getIndexBox()`メソッドを呼び出します。
*   `list()`メソッド:
    *   GuzzleHttpクライアントを使用してYouTube Data APIの`search`エンドポイントを呼び出し、ユーザーのクエリに基づいて動画IDを取得します。
    *   次に、取得した動画IDを使用して`videos`エンドポイントを呼び出し、動画の詳細情報（スニペット、統計情報など）を取得します。
    *   取得した動画情報をLaravel Adminの`Box`ウィジェットとBladeテンプレート（`exment_you_tube_search::list`）で表示します。
*   `save()`メソッド:
    *   リクエストから動画の情報を取得し、Exmentの`youtube`カスタムテーブルに新しいレコードとして保存します。
    *   保存成功のメッセージを表示し、前のページに戻ります。
*   `getIndexBox()`メソッド: YouTube検索フォームを含む`Box`ウィジェットを返します。このフォームは、YouTubeのアクセスキーが設定されているかどうかに応じて表示を切り替えます。
*   `setCustomOptionForm(&$form)`メソッド: プラグインの設定画面に、YouTube Data APIのアクセスキーを入力するためのテキストフィールドを追加します。

**このプラグインから学べること**:

*   Exmentにおけるページプラグインの基本的な構造と`config.json`でのルーティング設定方法。
*   `PluginPageBase`を継承したクラスでのカスタムページの構築方法。
*   GuzzleHttpクライアントを使用して外部API（YouTube Data API）と連携し、データを取得する方法。
*   取得したデータをExmentのカスタムテーブルに保存する方法。
*   `$useCustomOption`と`setCustomOptionForm()`メソッドを使用して、プラグインにカスタム設定画面を追加し、APIキーなどの設定を管理する方法。
*   Laravel AdminのウィジェットとBladeテンプレートを組み合わせて、複雑なUIを構築する方法。




### スクリプトプラグインのサンプル: CalendarBind

**概要**: `CalendarBind`は、Exmentのカレンダービューで、特定の条件（この例では「お知らせ」テーブルの「重要度」が4以上）に基づいてアイテムの色を動的に変更するスクリプトプラグインです。主にJavaScriptで実装され、フロントエンドの表示をカスタマイズします。

**`config.json`の要約**:

```json
{
    "uuid": "98cf18f5-7557-6d02-2655-bc6267c39c56",
    "plugin_name": "CalendarBind",
    "plugin_view_name": "カレンダービュー 条件によって色を変更",
    "description": "カレンダービューで、条件によって色を変更するサンプルです。",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "script"
}
```

*   `plugin_type`: `script`として定義されており、スクリプトプラグインであることを示します。

**`script.js`の要約**:

```javascript
$(function () {
    $(window).off(\'exment:calendar_bind\', scriptCalendarBind).on(\'exment:calendar_bind\', scriptCalendarBind);

    function scriptCalendarBind(e, event){
        // 「お知らせ」テーブルでないと終了
        // 前半：テーブルビューのチェック、後半：ダッシュボードのチェック
        if(!$(\".custom_value_information\").length && !$(\"[data-target_table_name=\"information\"]\").length){
            return;
        }

        // 「重要度」が高い(4)以上の場合は強制的に赤背景に変更
        if(event.value && event.value.priority >= 4){
            event.color = \'red\';
            event.textColor = \'white\';
        }
        return event;
    }
});
```

*   `exment:calendar_bind`イベントに`scriptCalendarBind`関数をバインドしています。
*   `scriptCalendarBind`関数は、イベントが「お知らせ」テーブルに関連しているかどうかを確認します。
*   もしイベントが「お知らせ」テーブルに関連しており、かつイベントの`priority`（重要度）が4以上の場合、イベントの背景色を赤に、文字色を白に強制的に変更します。

**このプラグインから学べること**:

*   Exmentにおけるスクリプトプラグインの基本的な構造と`config.json`での設定方法。
*   JavaScriptを使用してExmentのフロントエンドイベント（`exment:calendar_bind`）をリッスンし、カレンダービューの表示を動的に変更する方法。
*   特定のテーブルや条件に基づいて表示をカスタマイズする方法。
*   Exmentのフロントエンドで利用可能なイベントと、それらを活用したUIの拡張方法。




### スクリプトプラグインのサンプル: ChangeDynamicForm

**概要**: `ChangeDynamicForm`は、Exmentの入力フォームにおいて、選択された値に応じて他の項目の表示/非表示を動的に切り替えるスクリプトプラグインです。これにより、ユーザーインターフェースの使いやすさを向上させ、入力ミスを減らすことができます。

**`config.json`の要約**:

```json
{
    "uuid": "926deea0-0d56-11ed-aa05-0800200c9a66",
    "plugin_name": "ChangeDynamicForm",
    "plugin_view_name": "入力フォーム 動的切り替え",
    "description": "入力フォームの選択値に応じて、項目の表示非表示を切り替えます。",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "script"
}
```

*   `plugin_type`: `script`として定義されており、スクリプトプラグインであることを示します。

**`script.js`の要約**:

```javascript
$(function () {
    $(window).off(\'exment:form_loaded\', setDynamicFormEvent).on(\'exment:form_loaded\', setDynamicFormEvent);

    function setDynamicFormEvent(){
        setDynamicForm();
        // block_custom_value_formはデータ入力画面、custom_value_agencyは代理店テーブルの関連画面
        $(\'.block_custom_value_form\' + \'.custom_value_agency\')
            .off(\'change.exment_change_agency\', \".value_syubetu,.value_tantosha\", setDynamicForm)
            .on(\'change.exment_change_agency\', \".value_syubetu,.value_tantosha\", setDynamicForm)
    }

    function setDynamicForm(){
        // 種別と担当者の値を取得します。
        const value_syubetu = $(\'.value_syubetu\').val();
        const value_tantosha = $(\'.value_tantosha\').val();

        // 種別が法人の場合だけ法人コードを表示します。
        if(value_syubetu && value_syubetu.indexOf(\'法人\') < 0) {
            $(\'.value_corpcode\').closest(\'.row\').hide();
            $(\'.value_corpcode\').val(\'\');
        } else {
            $(\'.value_corpcode\').closest(\'.row\').show();
        }

        // 担当者が1（管理者）の場合だけインセンティブレートを表示します。
        if (value_tantosha == \'1\') {
            $(\'.value_Incentive\').closest(\'.row\').show();
        } else {
            $(\'.value_Incentive\').closest(\'.row\').hide();
            $(\'.value_Incentive\').val(\'\');
        }
    }
});
```

*   `exment:form_loaded`イベントに`setDynamicFormEvent`関数をバインドし、フォームがロードされた際に動的な表示切り替えを設定します。
*   `setDynamicFormEvent`関数は、`value_syubetu`（種別）と`value_tantosha`（担当者）の変更イベントを監視し、`setDynamicForm`関数を呼び出します。
*   `setDynamicForm`関数は、`value_syubetu`が「法人」を含まない場合に`value_corpcode`（法人コード）フィールドを非表示にし、`value_tantosha`が「1」（管理者）の場合に`value_Incentive`（インセンティブレート）フィールドを表示します。

**このプラグインから学べること**:

*   Exmentにおけるスクリプトプラグインの基本的な構造と`config.json`での設定方法。
*   JavaScriptとjQueryを使用して、Exmentの入力フォームの要素を動的に操作する方法。
*   特定のフィールドの値に基づいて、他のフィールドの表示/非表示を切り替えるロジックの実装方法。
*   `exment:form_loaded`イベントを利用して、フォームが完全にロードされた後にスクリプトを実行する方法。




### スクリプトプラグインのサンプル: ReplaceKanaHanZen

**概要**: `ReplaceKanaHanZen`は、Exmentのテキスト入力フィールドに入力された半角カナを自動的に全角カナに変換するスクリプトプラグインです。これにより、データの統一性を保ち、入力の手間を軽減します。

**`config.json`の要約**:

```json
{
    "uuid": "e2c18434-4dd6-4146-aa15-3b8003d4a77b",
    "plugin_name": "ReplaceKanaHanZen",
    "plugin_view_name": "カナ半角全角変換",
    "description": "半角のカナを全角に置き換えます。",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "script"
}
```

*   `plugin_type`: `script`として定義されており、スクリプトプラグインであることを示します。

**`script.js`の要約**:

```javascript
$(function () {
    $(window).off(\'exment:form_loaded\', setReplaceKanaHanZenEvent).on(\'exment:form_loaded\', setReplaceKanaHanZenEvent);

    function setReplaceKanaHanZenEvent(){
        $(".block_custom_value_form input[type=\'text\']").off(\'change\', replaceKanaHanZen).on(\'change\', replaceKanaHanZen);
    }

    function replaceKanaHanZen(ev){
        var $target = $(ev.target);
        if(!$target || $target.length == 0){
            return;
        }
        
        var str = $target.val();
        if(!str){
            return;
        }
        
            
        const kanaMap = {
            \'ｶﾞ\': \'ガ\', \'ｷﾞ\': \'ギ\', \'ｸﾞ\': \'グ\', \'ｹﾞ\': \'ゲ\', \'ｺﾞ\': \'ゴ\',
            \'ｻﾞ\': \'ザ\', \'ｼﾞ\': \'ジ\', \'ｽﾞ\': \'ズ\', \'ｾﾞ\': \'ゼ\', \'ｿﾞ\': \'ゾ\',
            \'ﾀﾞ\': \'ダ\', \'ﾁﾞ\': \'ヂ\', \'ﾂﾞ\': \'ヅ\', \'ﾃﾞ\': \'デ\', \'ﾄﾞ\': \'ド\',
            \'ﾊﾞ\': \'バ\', \'ﾋﾞ\': \'ビ\', \'ﾌﾞ\': \'ブ\', \'ﾍﾞ\': \'ベ\', \'ﾎﾞ\': \'ボ\',
            \'ﾊﾟ\': \'パ\', \'ﾋﾟ\': \'ピ\', \'ﾌﾟ\': \'プ\', \'ﾍﾟ\': \'ペ\', \'ﾎﾟ\': \'ポ\',
            \'ｳﾞ\': \'ヴ\', \'ﾜﾞ\': \'ヷ\', \'ｦﾞ\': \'ヺ\',
            \'ｱ\': \'ア\', \'ｲ\': \'イ\', \'ｳ\': \'ウ\', \'ｴ\': \'エ\', \'ｵ\': \'オ\',
            \'ｶ\': \'カ\', \'ｷ\': \'キ\', \'ｸ\': \'ク\', \'ｹ\': \'ケ\', \'ｺ\': \'コ\',
            \'ｻ\': \'サ\', \'ｼ\': \'シ\', \'ｽ\': \'ス\', \'ｾ\': \'セ\', \'ｿ\': \'ソ\',
            \'ﾀ\': \'タ\', \'ﾁ\': \'チ\', \'ﾂ\': \'ツ\', \'ﾃ\': \'テ\', \'ﾄ\': \'ト\',
            \'ﾅ\': \'ナ\', \'ﾆ\': \'ニ\', \'ﾇ\': \'ヌ\', \'ﾈ\': \'ネ\', \'ﾉ\': \'ノ\',
            \'ﾊ\': \'ハ\', \'ﾋ\': \'ヒ\', \'ﾌ\': \'フ\', \'ﾍ\': \'ヘ\', \'ﾎ\': \'ホ\',
            \'ﾏ\': \'マ\', \'ﾐ\': \'ミ\', \'ﾑ\': \'ム\', \'ﾒ\': \'メ\', \'ﾓ\': \'モ\',
            \'ﾔ\': \'ヤ\', \'ﾕ\': \'ユ\', \'ﾖ\': \'ヨ\',
            \'ﾗ\': \'ラ\', \'ﾘ\': \'リ\', \'ﾙ\': \'ル\', \'ﾚ\': \'レ\', \'ﾛ\': \'ロ\',
            \'ﾜ\': \'ワ\', \'ｦ\': \'ヲ\', \'ﾝ\': \'ン\',
            \'ｧ\': \'ァ\', \'ｨ\': \'ィ\', \'ｩ\': \'ゥ\', \'ｪ\': \'ェ\', \'ｫ\': \'ォ\',
            \'ｯ\': \'ッ\', \'ｬ\': \'ャ\', \'ｭ\': \'ュ\', \'ｮ\': \'ョ\',
            \'｡\': \'。\', \'､\': \'、\', \'ｰ\': \'ー\', \'｢\': \'「\', \'｣\': \'」\', \'･\': \'・\'
        };

        var reg = new RegExp(\'(\' + Object.keys(kanaMap).join(\'|\') + \')\', \'g\');
        str = str
            .replace(reg, function (match) {
                return kanaMap[match];
            })
            .replace(/ﾞ/g, \'゛\')
            .replace(/ﾟ/g, \'゜\');
        $target.val(str);
    }

});
```

*   `exment:form_loaded`イベントに`setReplaceKanaHanZenEvent`関数をバインドし、フォームがロードされた際にテキスト入力フィールドの`change`イベントを監視します。
*   `replaceKanaHanZen`関数は、入力フィールドの値が変更された際に、定義された`kanaMap`を使用して半角カナを全角カナに変換します。

**このプラグインから学べること**:

*   Exmentにおけるスクリプトプラグインの基本的な構造と`config.json`での設定方法。
*   JavaScriptとjQueryを使用して、Exmentの入力フォームのテキストフィールドの値を動的に変換する方法。
*   `exment:form_loaded`イベントを利用して、フォームが完全にロードされた後にスクリプトを実行する方法。
*   正規表現とマップを使用して、文字列の置換処理を効率的に行う方法。


### スクリプトプラグインのサンプル: ReplaceZenHan

**概要**: `ReplaceZenHan`は、Exmentのテキスト入力フィールドに入力された全角英数字を自動的に半角英数字に変換するスクリプトプラグインです。これにより、データの統一性を保ち、入力の手間を軽減します。

**`config.json`の要約**:

```json
{
    "uuid": "e2c18434-4dd6-4146-aa15-3b8003d4a77a",
    "plugin_name": "ReplaceZenHan",
    "plugin_view_name": "全角半角変換",
    "description": "全角の英数字を半角に置き換えます。",
    "author":  "Kajitori",
    "version": "1.0.1",
    "plugin_type": "script"
}
```

*   `plugin_type`: `script`として定義されており、スクリプトプラグインであることを示します。

**`script.js`の要約**:

```javascript
$(function () {
    $(window).off(\'exment:form_loaded\', setReplaceZenHanEvent).on(\'exment:form_loaded\', setReplaceZenHanEvent);

    function setReplaceZenHanEvent(){
        $(\'.block_custom_value_form\').off(\'change.exment_replace_zen_han\', \"input[type=\'text\']\", replaceZenHan).on(\'change.exment_replace_zen_han\', \"input[type=\'text\']\", replaceZenHan)
    }

    function replaceZenHan(ev){
        var $target = $(ev.target);
        if(!$target || $target.length == 0){
            return;
        }
        
        var str = $target.val();
        if(!str){
            return;
        }
        
        
        str = str.replace(/[Ａ-Ｚａ-ｚ０-９]/g, function(s) {
            return String.fromCharCode(s.charCodeAt(0) - 65248);
        });
        $target.val(str);
    }
});
```

*   `exment:form_loaded`イベントに`setReplaceZenHanEvent`関数をバインドし、フォームがロードされた際にテキスト入力フィールドの`change`イベントを監視します。
*   `replaceZenHan`関数は、入力フィールドの値が変更された際に、全角英数字を半角英数字に変換します。

**このプラグインから学べること**:

*   Exmentにおけるスクリプトプラグインの基本的な構造と`config.json`での設定方法。
*   JavaScriptとjQueryを使用して、Exmentの入力フォームのテキストフィールドの値を動的に変換する方法。
*   `exment:form_loaded`イベントを利用して、フォームが完全にロードされた後にスクリプトを実行する方法。
*   正規表現と`String.fromCharCode`を使用して、全角文字を半角文字に変換する方法。




### スクリプトプラグインのサンプル: SetAddress

**概要**: `SetAddress`は、Exmentの入力フォームにおいて、郵便番号を入力すると自動的に住所を補完するスクリプトプラグインです。`AjaxZip3`ライブラリを利用して、ユーザーの入力負担を軽減します。

**`config.json`の要約**:

```json
{
    "uuid": "b5c0a5d2-2716-1937-98d0-b490c1ebc533",
    "plugin_name": "SetAddress",
    "plugin_view_name": "入力フォーム 住所セット",
    "description": "入力フォームの郵便番号を使用し、住所をセットします。",
    "author":  "Kajitori",
    "version": "1.0.1",
    "plugin_type": "script"
}
```

*   `plugin_type`: `script`として定義されており、スクリプトプラグインであることを示します。

**`script.js`の要約**:

```javascript
$(function () {
    $(window).off(\'exment:form_loaded\', setAddressEvent).on(\'exment:form_loaded\', setAddressEvent);

    function setAddressEvent(){
        $(\'.block_custom_value_form\').off(\'change.exment_set_address\', \".value_zip01,.value_zip02\", setAddressKeyup).on(\'change.exment_set_address\', \".value_zip01,.value_zip02\", setAddressKeyup)
    }

    function setAddressKeyup(){
        const value_zip01 = $(\'.value_zip01\').val();
        const value_zip02 = $(\'.value_zip02\').val();

        if(!value_zip01 || !value_zip02){
            $(\'.value_addr01\').val(\'\');
            $(\'.value_addr02\').val(\'\');
            $(\'.value_pref\').val(\'\');
            return;
        }

        AjaxZip3.zip2addr(\'value[zip01]\',\'value[zip02]\',\'value[pref]\',\'value[addr01]\',\'value[addr02]\');
    }
});
```

*   `exment:form_loaded`イベントに`setAddressEvent`関数をバインドし、フォームがロードされた際に郵便番号フィールド（`value_zip01`, `value_zip02`）の`change`イベントを監視します。
*   `setAddressKeyup`関数は、郵便番号が入力された際に`AjaxZip3.zip2addr`関数を呼び出し、住所フィールド（`value_pref`, `value_addr01`, `value_addr02`）を自動的に補完します。

**このプラグインから学べること**:

*   Exmentにおけるスクリプトプラグインの基本的な構造と`config.json`での設定方法。
*   JavaScriptとjQueryを使用して、Exmentの入力フォームのフィールドを動的に操作する方法。
*   `exment:form_loaded`イベントを利用して、フォームが完全にロードされた後にスクリプトを実行する方法。
*   外部JavaScriptライブラリ（`AjaxZip3`）をExmentプラグイン内で利用し、機能を追加する方法。




### スクリプトプラグインのサンプル: SetStyle

**概要**: `SetStyle`は、ExmentのUI全体にカスタムスタイルを適用するスタイルプラグインです。このサンプルでは、すべての文字色を赤色に変更するCSSを適用します。

**`config.json`の要約**:

```json
{
    "uuid": "b5c0a5d2-2716-1937-98d0-b490c1ebc544",
    "plugin_name": "set_sytle",
    "plugin_view_name": "スタイルテスト",
    "description": "スタイルテストです。すべての文字色を赤色にします。",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "style"
}
```

*   `plugin_type`: `style`として定義されており、スタイルプラグインであることを示します。

**`style.css`の要約**:

```css
*{
    color:red !important;
}
```

*   このCSSは、ページ内のすべての要素の文字色を赤色に強制的に変更します。

**このプラグインから学べること**:

*   Exmentにおけるスタイルプラグインの基本的な構造と`config.json`での設定方法。
*   CSSファイルを使用してExmentのUIにカスタムスタイルを適用する方法。
*   `!important`キーワードを使用して、既存のスタイルを上書きする方法。




### バリデーションプラグインのサンプル: PluginValidatorTest

**概要**: `PluginValidatorTest`は、Exmentのカスタムテーブルのデータ入力時に、独自のバリデーションルールを適用するプラグインです。このサンプルでは、「information」テーブルの「order」列の値が100を超えないようにバリデーションを行います。

**`config.json`の要約**:

```json
{
    "plugin_name": "PluginValidatorTest",
    "uuid": "63d80590-4aad-11e9-b475-0800200c9a67",
    "plugin_view_name": "バリデーションテスト",
    "description": "カスタムテーブルのバリデーションを行います。",
    "author": "Kajitori",
    "version": "1.0.0",
    "target_tables": "information",
    "plugin_type": "validator"
}
```

*   `plugin_type`: `validator`として定義されており、バリデーションプラグインであることを示します。
*   `target_tables`: `information`に設定されており、`information`テーブルのデータが対象であることを示します。

**`Plugin.php`の要約**:

```php
<?php
namespace App\Plugins\PluginValidatorTest;

use Exceedone\Exment\Services\Plugin\PluginValidatorBase;
class Plugin extends PluginValidatorBase
{
    /**
     * Plugin Validator
     */
    public function validate()
    {
        // 入力値を取得する
        $priority = array_get($this->input_value, \'order\');

        if ($priority > 100) {
            // エラーメッセージを設定する（キー：列名、値：メッセージ）
            $this->messages[\'order\'] = \'表示順には100以下の数値を入れてください！\';
            // 戻り値にfalse（エラー発生）を返す
            return false;
        }
        // 戻り値にtrue（正常）を返す
        return true;
    }
}
```

*   `PluginValidatorBase`を継承しており、バリデーションプラグインの基底機能を提供します。
*   `validate()`メソッド:
    *   `$this->input_value`から入力値（この例では`order`列の値）を取得します。
    *   `order`の値が100を超えている場合、`$this->messages`配列にエラーメッセージを設定し、`false`を返してバリデーションエラーを示します。
    *   それ以外の場合は`true`を返してバリデーション成功を示します。

**このプラグインから学べること**:

*   Exmentにおけるバリデーションプラグインの基本的な構造と`config.json`での設定方法。
*   `PluginValidatorBase`を継承したクラスでのカスタムバリデーションロジックの実装方法。
*   `$this->input_value`を使用して入力値にアクセスする方法。
*   `$this->messages`配列にエラーメッセージを設定し、ユーザーに表示する方法。
*   `validate()`メソッドの戻り値（`true`/`false`）によってバリデーションの成否を制御する方法。




### ビュープラグインのサンプル: KanbanView

**概要**: `KanbanView`は、Exmentのデータをシンプルなかんばん形式で表示するビュープラグインです。カテゴリ列に基づいてボードを生成し、ドラッグ＆ドロップでアイテムのカテゴリを変更できる機能を提供します。

**`config.json`の要約**:

```json
{
    "plugin_name": "KanbanView",
    "plugin_view_name" : "かんばんビュー",
    "explain": "シンプルなかんばんビューを表示するプラグインです。",
    "uuid":  "648c8d9e-dd0d-4926-9bde-968f6c748abc",
    "author":  "Kajitori",
    "version": "0.0.1",
    "plugin_type": "view",
    "route": [
        {
            "uri": "update",
            "method": [
                "post"
            ],
            "function": "update"
        }
    ]
}
```

*   `plugin_type`: `view`として定義されており、ビュープラグインであることを示します。
*   `route`: `update`というURIでPOSTリクエストを受け付け、`update`メソッドを実行するルートが定義されています。これはかんばんボード上でのアイテムの移動（カテゴリ変更）に使用されます。

**`Plugin.php`の要約**:

```php
<?php

namespace App\Plugins\KanbanView;

use Exceedone\Exment\Services\Plugin\PluginViewBase;
use Exceedone\Exment\Enums\ColumnType;
use Exceedone\Exment\Model\CustomTable;
use Exceedone\Exment\Model\CustomColumn;

class Plugin extends PluginViewBase
{
    /**
     *
     */
    public function grid()
    {
        $values = $this->values();
        return $this->pluginView(\'kanban\', [\'values\' => $values]);
    }

    /**
     * (2) このプラグイン独自のエンドポイント
     */
    public function update(){
        $value = request()->get(\'value\');

        $custom_table = CustomTable::getEloquent(request()->get(\'table_name\'));
        $custom_value = $custom_table->getValueModel(request()->get(\'id\'));

        $custom_value->setValue($value)
            ->save();

        return response()->json($custom_value);
    }
    

    /**
     * Set view option form for setting
     *
     * @param Form $form
     * @return void
     */
    public function setViewOptionForm($form)
    {
        //　独自設定を追加する場合
        $form->embeds(\'custom_options\', \'詳細設定\', function($form){
            $form->select(\'category\', \'カテゴリ列\')
                ->options($this->custom_table->getFilteredTypeColumns([ColumnType::SELECT, ColumnType::SELECT_VALTEXT])->pluck(\'column_view_name\', \'id\'))
                ->required()
                ->help(\'カテゴリ列を選択してください。カンバンのボードに該当します。カスタム列種類「選択肢」「選択肢(値・見出し)」が候補に表示されます。\');
        });

        //　フィルタ(絞り込み)の設定を行う場合
        static::setFilterFields($form, $this->custom_table);

        // 並べ替えの設定を行う場合
        static::setSortFields($form, $this->custom_table);
    }
    

    protected function values(){
        $query = $this->custom_table->getValueQuery();

        // データのフィルタを実施
        $this->custom_view->filterModel($query);

        // データのソートを実施
        $this->custom_view->sortModel($query);

        // 値を取得
        $items = collect();
        $query->chunk(1000, function($values) use(&$items){
            $items = $items->merge($values);
        });

        $boards = $this->getBoardItems($items);

        return $boards;
    }


    protected function getBoardItems($items){
        $category = CustomColumn::getEloquent($this->custom_view->getCustomOption(\'category\'));
        $options = $category->createSelectOptions();

        $update_url = $this->plugin->getFullUrl(\'update\');

        // set boards
        $boards_dragTo = collect($options)->map(function($option, $key){
            return \"board-id-$key\";
        })->toArray();

        $boards = collect($options)->map(function($option, $key) use($category, $boards_dragTo){
            return [
                \'id\' => \"board-id-$key\";
                \'column_name\' => $category->column_name,
                \'key\' => $key,
                \'title\' => $option,
                \'drapTo\' => $boards_dragTo,
                \'item\' => [],
            ];
        })->values()->toArray();

        foreach($items as $item){
            $c = array_get($item, \'value.\' . $category->column_name);
            
            foreach($boards as &$board){
                if(!isMatchString($c, $board[\'key\'])){
                    continue;
                }

                $board[\'item\'][] = [
                    \'id\' => \"item-id-$item->id\";
                    \'title\' => $item->getLabel(),
                    \'dataid\' => $item->id,
                    \'table_name\' => $this->custom_table->table_name,
                    \'update_url\' => $update_url,
                ];
            }
        }

        return $boards;
    }
}
```

*   `PluginViewBase`を継承しており、ビュープラグインの基底機能を提供します。
*   `grid()`メソッド: かんばんビューの表示ロジックを実装しています。`values()`メソッドでデータを取得し、`kanban`ブレードテンプレートに渡してレンダリングします。
*   `update()`メソッド: かんばんボード上でアイテムが移動された際に呼び出されます。リクエストからテーブル名、アイテムID、新しいカテゴリの値を取得し、Exmentのカスタム値モデルを更新してデータを保存します。
*   `setViewOptionForm($form)`メソッド: ビューのオプション設定フォームを定義します。ここでは、かんばんボードのカテゴリとして使用する列を選択するためのドロップダウン（`category`）を追加します。この列は「選択肢」または「選択肢(値・見出し)」タイプのカスタム列である必要があります。
*   `values()`メソッド: カスタムテーブルからデータを取得し、フィルタリングとソートを適用します。その後、`getBoardItems()`メソッドを呼び出して、かんばんボードの形式にデータを整形します。
*   `getBoardItems($items)`メソッド: 選択されたカテゴリ列に基づいてかんばんボードの構造を生成します。各ボードには、そのカテゴリに属するアイテムが格納されます。アイテムのドラッグ＆ドロップを可能にするための情報も設定されます。

**このプラグインから学べること**:

*   Exmentにおけるビュープラグインの基本的な構造と`config.json`でのルーティング設定方法。
*   `PluginViewBase`を継承したクラスでのカスタムビューの構築方法。
*   `setViewOptionForm()`メソッドを使用して、ビューに独自のオプション設定を追加する方法。
*   Exmentのカスタムテーブルからデータを取得し、カスタム列の情報を利用してデータを整形する方法。
*   ドラッグ＆ドロップなどのインタラクティブなUIを実装するためのバックエンドロジックの例。
*   Bladeテンプレートと連携して動的なビューを生成する方法。




### スクリプトプラグインのサンプル: TestScript

**概要**: `TestScript`は、Exmentの各種イベント（ダッシュボードロード時、ページロード時など）をフックしてJavaScriptを実行するスクリプトプラグインです。主に開発やデバッグ目的で、イベントの発生を確認したり、コンソールにメッセージを出力したりするのに使用されます。

**`config.json`の要約**:

```json
{
    "uuid": "b5c0a5d2-2716-1937-98d0-b490c1ebc511",
    "plugin_name": "TestScript",
    "plugin_view_name": "スクリプトテスト",
    "description": "一通りのスクリプトをテストします。",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "script"
}
```

*   `plugin_type`: `script`として定義されており、スクリプトプラグインであることを示します。

**`script.js`の要約**:

```javascript
$(function () {
    $(window).off(\'exment:dashboard_loaded\', \'.box-dashboard\', {}, scriptTestDashboard).on(\'exment:dashboard_loaded\', \'.box-dashboard\', {}, scriptTestDashboard);
    $(window).off(\'exment:loaded\', scriptTest).on(\'exment:loaded\', scriptTest);

    function scriptTestDashboard(ev){
        console.log(ev.target);
    }

    function scriptTest(){
        console.log(\'loaded test\');
    }
});
```

*   `exment:dashboard_loaded`イベントに`scriptTestDashboard`関数をバインドしています。このイベントはダッシュボードがロードされた際に発火し、イベントターゲットをコンソールに出力します。
*   `exment:loaded`イベントに`scriptTest`関数をバインドしています。このイベントはExmentのページがロードされた際に発火し、「loaded test」というメッセージをコンソールに出力します。

**このプラグインから学べること**:

*   Exmentにおけるスクリプトプラグインの基本的な構造と`config.json`での設定方法。
*   JavaScriptとjQueryを使用して、Exmentの特定のイベント（`exment:dashboard_loaded`, `exment:loaded`）をリッスンする方法。
*   イベントハンドラ内でコンソールにメッセージを出力したり、イベントオブジェクトにアクセスしたりする方法。
*   Exmentのフロントエンド開発において、デバッグやイベントの動作確認を行うための基本的なアプローチ。




## ビュープラグインにおける `public function setViewOptionForm($form)` の実装例と解説

Exmentのビュープラグインにおいて、`public function setViewOptionForm($form)` メソッドは、ユーザーがビューの表示設定をカスタマイズするためのフォームを定義するために使用されます。このメソッドを通じて、ビューの動作に影響を与える様々なオプション（例: どの列をタイトルとして使用するか、どの列を開始日として使用するかなど）をユーザーが選択できるようにします。

### `setViewOptionForm` の機能と目的

`setViewOptionForm` メソッドは、`PluginViewBase` クラスを継承したビュープラグインに実装されます。このメソッドは、Laravel Admin のフォームビルダーインスタンス `$form` を引数として受け取ります。開発者はこの `$form` オブジェクトを使用して、ビューの設定に必要な入力フィールド（セレクトボックス、テキスト入力、チェックボックスなど）を動的に追加できます。

主な目的は以下の通りです。

1.  **ユーザーカスタマイズの提供**: ビューの表示や動作をユーザーが柔軟に設定できるようにします。
2.  **設定の永続化**: ユーザーが設定したオプションをExmentのデータベースに保存し、次回ビューを開いた際にその設定が反映されるようにします。
3.  **動的なフォーム生成**: データベースのスキーマ情報（カスタムテーブルの列など）に基づいて、適切な選択肢や入力フィールドを動的に生成します。
4.  **バリデーションとヘルプテキスト**: 各設定項目に対して、必須入力の指定や、ユーザーが設定内容を理解するためのヘルプテキストを提供します。


```php
public function setViewOptionForm($form)
{
    // 独自設定を追加する場合
    $form->embeds('custom_options', '詳細設定', function($form) {
        $form->select('title_column', 'タイトル列')
            ->options($this->custom_table->custom_columns->pluck('column_view_name', 'id'))
            ->help('タスクのタイトルとして表示する列を選択してください。選択しない場合はデフォルトのラベルが使用されます。');
            
        $form->select('start_date_column', '開始日列')
            ->options($this->custom_table->getFilteredTypeColumns([ColumnType::DATE, ColumnType::DATETIME])->pluck('column_view_name', 'id'))
            ->required()
            ->help('タスクの開始日となる列を選択してください。カスタム列種類「日付」「日時」が候補に表示されます。');
            
        $form->select('end_date_column', '終了日列')
            ->options($this->custom_table->getFilteredTypeColumns([ColumnType::DATE, ColumnType::DATETIME])->pluck('column_view_name', 'id'))
            ->required()
            ->help('タスクの終了日となる列を選択してください。カスタム列種類「日付」「日時」が候補に表示されます。');
            
        $form->select('progress_column', '進捗率列')
            ->options($this->custom_table->getFilteredTypeColumns([ColumnType::INTEGER, ColumnType::DECIMAL,ColumnType::SELECT_VALTEXT])->pluck('column_view_name', 'id'))
            ->help('タスクの進捗率を表す列を選択してください。カスタム列種類「整数」「小数」「選択肢(値・見出し)」が候補に表示されます。');
            
        $form->select('color_column', '色指定列')
            ->options($this->custom_table->custom_columns->pluck('column_view_name', 'id'))
            ->help('タスクの色を指定する列を選択してください。列の値が「赤」「青」「緑」の場合、対応する色でタスクが表示されます。それ以外の値や値がない場合は青色で表示されます。');
    });
    
    // フィルタ(絞り込み)の設定を行う場合
    static::setFilterFields($form, $this->custom_table);
    
    // 並べ替えの設定を行う場合
    static::setSortFields($form, this->custom_table);
}
```

#### 1. `$form->embeds('custom_options', '詳細設定', function($form) { ... });`

*   **`embeds`**: これは、フォーム内にネストされたセクションを作成するために使用されます。`custom_options`というキーで設定が保存され、UI上では「詳細設定」というラベルで表示されます。
*   **目的**: ビュー固有のカスタム設定をグループ化し、他の共通設定（フィルタや並べ替え）と区別するために利用されます。これにより、設定画面が整理され、ユーザーにとって分かりやすくなります。

#### 2. 各 `select` フィールドの定義

各 `select` フィールドは、Exmentのカスタムテーブルの特定の列を選択するためのドロップダウンメニューを生成します。

*   **`$form->select('field_name', '表示ラベル')`**: ドロップダウンメニューを定義します。`field_name`は設定が保存される際のキーとなり、`表示ラベル`はUIに表示されるユーザー向けの名称です。

*   **`->options(...)`**: ドロップダウンの選択肢を定義します。ここでは、`$this->custom_table` オブジェクトから利用可能なカスタム列のリストを動的に取得しています。
    *   `$this->custom_table->custom_columns->pluck('column_view_name', 'id')`: これは、現在のカスタムテーブルのすべてのカスタム列を取得し、その表示名（`column_view_name`）を値、ID（`id`）をキーとする連想配列を生成します。これにより、ユーザーは列の表示名を見て選択し、プラグインは選択された列のIDで処理を行うことができます。
    *   `$this->custom_table->getFilteredTypeColumns([ColumnType::DATE, ColumnType::DATETIME])->pluck('column_view_name', 'id')`: 特定のデータ型（例: `ColumnType::DATE`, `ColumnType::DATETIME`）にフィルタリングされたカスタム列のみを選択肢として表示します。これにより、ユーザーは適切なデータ型の列のみを選択でき、設定ミスを防ぐことができます。

*   **`->required()`**: この設定項目が必須であることを示します。ユーザーがこの項目を選択しないと、設定を保存できません。

*   **`->help('ヘルプテキスト')`**: ユーザーが設定項目を理解するための説明文を提供します。これはUI上でツールチップや説明文として表示されます。

**各 `select` フィールドの具体的な役割**: 

*   **`title_column` (タイトル列)**: ビュー内で各アイテムのタイトルとして表示する列を選択します。例えば、タスク管理ビューであればタスク名を表示する列を指定します。
*   **`start_date_column` (開始日列)**: アイテムの開始日を示す列を選択します。ガントチャートやカレンダービューで必須となる情報です。`ColumnType::DATE`または`ColumnType::DATETIME`の列のみが選択肢となります。
*   **`end_date_column` (終了日列)**: アイテムの終了日を示す列を選択します。開始日列と同様に、タイムラインベースのビューで必須です。`ColumnType::DATE`または`ColumnType::DATETIME`の列のみが選択肢となります。
*   **`progress_column` (進捗率列)**: アイテムの進捗状況を示す列を選択します。数値型（`INTEGER`, `DECIMAL`）または選択肢型（`SELECT_VALTEXT`）の列が候補となります。
*   **`color_column` (色指定列)**: アイテムの表示色を決定する列を選択します。この例では、列の値が「赤」「青」「緑」の場合に特定の色を適用するロジックが想定されています。

#### 3. `static::setFilterFields($form, $this->custom_table);`
*   **目的**: これは、Exmentのビュープラグインに共通して提供されるフィルタ（絞り込み）設定のフォームを追加するためのヘルパーメソッドです。ユーザーはここで、ビューに表示するデータを特定の条件で絞り込むための設定を行うことができます。
*   **利点**: 開発者がフィルタ機能をゼロから実装する必要がなく、共通のUIとロジックを再利用できます。

#### 4. `static::setSortFields($form, $this->custom_table);`

*   **目的**: これは、Exmentのビュープラグインに共通して提供される並べ替え設定のフォームを追加するためのヘルパーメソッドです。ユーザーはここで、ビューに表示するデータの並べ替え順序（昇順/降順、対象列）を設定することができます。
*   **利点**: 開発者が並べ替え機能をゼロから実装する必要がなく、共通のUIとロジックを再利用できます。

### `setViewOptionForm` のベストプラクティスとコード例

`setViewOptionForm` を実装する際のベストプラクティスと、提供されたコードをより明確にした例を以下に示します。

#### ベストプラクティス
1.  **明確なラベルとヘルプテキスト**: 各設定項目には、ユーザーがその目的をすぐに理解できるような明確なラベルと、必要に応じて詳細なヘルプテキストを提供します。
2.  **適切な入力タイプ**: 設定内容に応じて、`select`（ドロップダウン）、`text`（テキスト入力）、`number`（数値入力）、`checkbox`（チェックボックス）など、適切なフォーム要素を選択します。
3.  **動的な選択肢の生成**: カスタムテーブルの列など、Exmentの既存データに基づいて選択肢を動的に生成することで、ユーザーの利便性を高め、設定ミスを減らします。
4.  **必須項目の指定**: ビューの機能に不可欠な設定項目には`->required()`を適用し、ユーザーが必須項目を設定し忘れることを防ぎます。
5.  **設定のグループ化**: `embeds` を使用して関連する設定項目をグループ化し、設定画面の視認性と整理を向上させます。
6.  **共通ヘルパーの活用**: `setFilterFields` や `setSortFields` のようなExmentが提供する共通ヘルパーメソッドを積極的に活用し、開発コストを削減し、一貫したユーザーエクスペリエンスを提供します。
7.  **エラーハンドリングとバリデーション**: `required()` などのLaravel Adminのバリデーション機能を利用し、不正な設定値が保存されないようにします。

#### コード例（提供されたものをより明確にコメント付けしたもの）

```php
<?php

namespace App\Plugins\YourPluginName; // あなたのプラグインのnamespaceに置き換えてください

use Exceedone\Exment\Services\Plugin\PluginViewBase;
use Exceedone\Exment\Enums\ColumnType; // ColumnType enumを使用するためにインポート
use Encore\Admin\Form; // Formクラスを使用するためにインポート

class Plugin extends PluginViewBase
{
    /**
     * ビューのオプション設定フォームを定義します。
     * ユーザーがこのビューの表示や動作をカスタマイズするための設定項目を提供します。
     *
     * @param Form $form Laravel Adminのフォームビルダーインスタンス
     * @return void
     */
    public function setViewOptionForm($form)
    {
        // 独自設定を「詳細設定」というグループで追加します。
        // これらの設定は、ビューのカスタムオプションとして保存されます。
        $form->embeds('custom_options', '詳細設定', function($form) {

            // タイトル列の選択: ビュー内でアイテムのタイトルとして表示する列を指定します。
            // すべてのカスタム列が選択肢として表示されます。
            $form->select('title_column', 'タイトル列')
                ->options($this->custom_table->custom_columns->pluck('column_view_name', 'id'))
                ->help('タスクのタイトルとして表示する列を選択してください。選択しない場合はデフォルトのラベルが使用されます。');
                
            // 開始日列の選択: アイテムの開始日を示す列を指定します。
            // 日付型または日時型のカスタム列のみが選択肢として表示され、この項目は必須です。
            $form->select('start_date_column', '開始日列')
                ->options($this->custom_table->getFilteredTypeColumns([ColumnType::DATE, ColumnType::DATETIME])->pluck('column_view_name', 'id'))
                ->required()
                ->help('タスクの開始日となる列を選択してください。カスタム列種類「日付」「日時」が候補に表示されます。');
                
            // 終了日列の選択: アイテムの終了日を示す列を指定します。
            // 日付型または日時型のカスタム列のみが選択肢として表示され、この項目は必須です。
            $form->select('end_date_column', '終了日列')
                ->options($this->custom_table->getFilteredTypeColumns([ColumnType::DATE, ColumnType::DATETIME])->pluck('column_view_name', 'id'))
                ->required()
                ->help('タスクの終了日となる列を選択してください。カスタム列種類「日付」「日時」が候補に表示されます。');
                
            // 進捗率列の選択: アイテムの進捗状況を示す列を指定します。
            // 整数型、小数型、または選択肢(値・見出し)型のカスタム列が選択肢として表示されます。
            $form->select('progress_column', '進捗率列')
                ->options($this->custom_table->getFilteredTypeColumns([ColumnType::INTEGER, ColumnType::DECIMAL, ColumnType::SELECT_VALTEXT])->pluck('column_view_name', 'id'))
                ->help('タスクの進捗率を表す列を選択してください。カスタム列種類「整数」「小数」「選択肢(値・見出し)」が候補に表示されます。');
                
            // 色指定列の選択: アイテムの表示色を決定する列を指定します。
            // すべてのカスタム列が選択肢として表示されます。この列の値に基づいて、ビュー側で色を適用するロジックを実装する必要があります。
            $form->select('color_column', '色指定列')
                ->options($this->custom_table->custom_columns->pluck('column_view_name', 'id'))
                ->help('タスクの色を指定する列を選択してください。列の値が「赤」「青」「緑」の場合、対応する色でタスクが表示されます。それ以外の値や値がない場合は青色で表示されます。');
        });
        
        // Exmentが提供する共通のフィルタ(絞り込み)設定フォームを追加します。
        // これにより、ユーザーはビューに表示するデータを柔軟に絞り込むことができます。
        static::setFilterFields($form, $this->custom_table);
        
        // Exmentが提供する共通の並べ替え設定フォームを追加します。
        // これにより、ユーザーはビューに表示するデータの並べ替え順序を設定することができます。
        static::setSortFields($form, $this->custom_table);
    }
}
```

この例は、ビュープラグインの設定フォームをどのように構築し、Exmentの既存のデータ構造と連携させるかを示しています。`setViewOptionForm` を適切に実装することで、ユーザーにとって非常に柔軟で使いやすいビューを提供することが可能になります。




## プラグイン(イベント)
Exmentの画面上で特定の操作を行った場合に実行され、値の更新などの処理を行うことができるプラグインです。この機能は、v3.2.0より追加されました。

### イベントの種類
Exmentで利用可能なイベントの種類とそれぞれの説明は以下の通りです。

| 名前 | 種類 | 説明 |
| --- | --- | --- |
| loading | ロード開始時 | ページのロード処理の最初に、処理が起動します。 |
| saving | 保存直前 | データの保存直前に、処理が起動します。 |
| saved | 保存後 | データの保存後に、処理が起動します。 |
| deleted | 削除後 | データの削除後に、処理が起動します。 |
| workflow_action_executing | ワークフロー実行直前 | ワークフロー実行の直前に、処理が起動します。 |
| workflow_action_executed | ワークフロー実行後 | ワークフロー実行後に、処理が起動します。 |
| notify_executing | 通知実行直前 | 通知実行の直前に、処理が起動します。 |
| notify_executed | 通知実行後 | 通知実行後に、処理が起動します。 |

### 作成方法

イベントプラグインを作成するには、主に以下の手順が必要です。

#### config.json作成

以下の形式で`config.json`ファイルを作成します。

```json
{
    "plugin_name": "PluginDemoEvent",
    "uuid": "fa7de170-992a-11e8-b568-0800200c9a66",
    "plugin_view_name": "Plugin Event",
    "description": "プラグインイベントを実行するテストです。",
    "author": "(Your Name)",
    "version": "1.0.0",
    "plugin_type": "event",
    "event_triggers": "loaded"
}
```

*   `plugin_name`: 半角英数でプラグイン名を記入します。
*   `uuid`: プラグインを一意にするための32文字の文字列（ハイフンを含めると36文字）です。UUIDジェネレーター（例: [https://www.famkruithof.net/uuid/uuidgen](https://www.famkruithof.net/uuid/uuidgen)）で作成してください。
*   `plugin_type`: `event`と記入します。
*   `event_triggers`: 最初からトリガーを指定する場合、「イベントの種類」の名称をカンマ区切りで入力します。
*   `target_tables`: 最初から対象テーブルを指定する場合、カスタムテーブルのテーブル名をカンマ区切りで入力します。

#### PHPファイル作成

`Plugin.php`という名前で以下のPHPファイルを作成します。

```php
<?php
namespace App\Plugins\PluginDemoEvent;

use Exceedone\Exment\Services\Plugin\PluginEventBase;
class Plugin extends PluginEventBase
{
    /**
     * Plugin Trigger
     */
    public function execute()
    {
        \Log::debug(\'Event called!\');
        return true;
    }
}
```

*   `namespace`は、`App\Plugins\(プラグイン名のパスカルケース)`としてください。
*   プラグイン管理画面で登録したトリガーの条件に合致した場合に、この`execute`関数が実行されます。
*   `Plugin`クラスは`PluginEventBase`を継承します。`PluginEventBase`は、呼び出し元のカスタムテーブル`$custom_table`やテーブル値`$custom_value`などのプロパティを所有しており、`execute`関数が呼び出された時点でそれらの値が代入されます。プロパティの詳細については、[プラグインリファレンス](https://exment.net/docs/#/ja/plugin_quickstart_view?id=plugin-reference)を参照してください。

#### zipに圧縮

作成した`config.json`と`Plugin.php`（およびその他の必要なファイル）を最小構成としてzipファイルに圧縮します。zipファイル名は「`(plugin_name).zip`」としてください。

例:
*   `PluginDemoEvent.zip`
    *   `config.json`
    *   `Plugin.php`
    *   (その他、必要なPHPファイル、画像ファイルなど)

### サンプルプラグイン (SyncCity)

*   このサンプルは、カスタムデータの保存時に外部データベースとの連携を行うイベントプラグインです。
*   Exmentの都市テーブルで追加・更新・削除を行うと、外部データベースの`city`テーブルにその変更が反映されます。
*   **事前準備**:
    1.  Exmentの管理者設定→テンプレートから、[テンプレート](https://exment.net/docs/#/ja/template)をインポートします。
    2.  外部データベースを作成します。このプラグインではMySQLのサンプルデータベース「world」を利用しています。[公式サイト](https://dev.mysql.com/doc/index-other.html)からzipをダウンロードし、解凍したSQLをお使いのMySQL（またはMariaDB）環境で実行してください。
    3.  Exmentの管理者設定→プラグインから、[プラグイン](https://exment.net/docs/#/ja/plugin_quickstart_view?id=plugin-installation)をアップロードします。
    4.  プラグインの設定画面を開き、手順2で設定した外部データベースの接続情報を入力し、保存してください。




## 追加のプラグインタイプ

### エクスポート

エクスポートプラグインは、カスタムデータ一覧のエクスポートを独自に実装したい場合に使用できます。オリジナルフォーマットのファイルをエクスポートする場合や、特殊な変換処理を実装する場合にご利用ください。

#### 出力形式について

出力形式は、Excelの他、それ以外のフォーマットでの出力も可能です。Excel形式の場合、処理をより最適化しております。

#### 作成方法（通常）

1. **config.json作成**

```json
{
    "plugin_name": "ExportTestCsv",
    "uuid": "1e7881d0-324f-11e9-b56e-0800200c9a33",
    "plugin_view_name": "ExportTestCsv",
    "description": "エクスポートのCSVテストです。",
    "author": "Kajitori",
    "version": "0.0.1",
    "plugin_type": "export",
    "target_tables": "information",
    "label": "お知らせ情報出力",
    "icon": "fa-question",
    "export_description": "お知らせ情報を、独自のcsv形式で一覧出力します。"
}
```

2. **PHPファイル作成**

```php
<?php
namespace App\Plugins\ExportTestCsv;

use Exceedone\Exment\Services\Plugin\PluginExportBase;

class Plugin extends PluginExportBase
{
    /**
     * execute
     */
    public function execute() 
    {
        // ※メソッド「$this->getTmpFullPath()」で、一時tmpファイルを取得する
        // ※実行後、一時tmpファイルは自動的に削除されます。
        $tmp = $this->getTmpFullPath();

        ///// 独自の実装処理---ここから
        // csvファイルを開く
        $fp = fopen($tmp, 'w');

        // すべてのシステム列・カスタム列でデータ一覧取得（配列）
        $data = $this->getData();

        // ビュー形式でデータ一覧取得（配列）
        // $data = $this->getViewData();

        // CustomValueのCollectionでデータ一覧取得
        // $data = $this->getRecords();

        foreach ($data as $fields) {
            fputcsv($fp, $fields);
        }

        fclose($fp);
        ///// 独自の実装処理---ここまで

        // $tmpのstring文字列を返却する
        return $tmp;
    }

    /**
     * Get download file name.
     * ファイル名を取得する
     *
     * @return string
     */
    public function getFileName() : string {
        return "test.csv";
    }
}
```

#### 作成方法（Excel）

Excel形式で出力する場合も基本的には同様の手順で実装しますが、PHPファイルの作成時、より便利に実装できます。PHPでExcelを操作するためのライブラリPhpSpreadsheetを使用しています。

```php
<?php
namespace App\Plugins\ExportTestExcel;

// PluginExportExcelに変更
use Exceedone\Exment\Services\Plugin\PluginExportExcel;

// extendsをPluginExportExcelに変更
class Plugin extends PluginExportExcel
{
    /**
     * execute
     */
    public function execute() {
        // テンプレートファイルを読み込み、PhpSpreadsheetを初期化
        $spreadsheet = $this->initializeExcel('template.xlsx');
        // ※テンプレートファイルを使用せず、新規にファイルを作成する場合
        // $spreadsheet = $this->initializeExcel();

        // CustomValueのCollectionでデータ一覧取得
        $data = $this->getRecords();

        ///// 独自の実装処理---ここから
        $sheet = $spreadsheet->getActiveSheet();
        $column = 3;
        foreach($data as $record){
            // データをループしてセット
            $sheet->setCellValue("A{$column}", $record->id); // ID
            $sheet->setCellValue("B{$column}", $record->getValue('title', true)); // タイトル
            $sheet->setCellValue("C{$column}", $record->getValue('priority', true)); // 重要度
            $sheet->setCellValue("D{$column}", $record->updated_at); // 更新日時

            $column++;
        }

        // 枠の設定
        $laseRow = $column - 1;
        $sheet->getStyle("A2:D{$laseRow}")->applyFromArray([
            'borders' => [
                'outside'=>[
                    'borderStyle'=>\PhpOffice\PhpSpreadsheet\Style\Border::BORDER_THIN
                ],
                'inside'=>[
                    'borderStyle'=>\PhpOffice\PhpSpreadsheet\Style\Border::BORDER_HAIR
                ],
            ],
        ]);

        // 印刷範囲の設定
        $sheet->getPageSetup()->setPrintArea("A1:D{$laseRow}");
        ///// 独自の実装処理---ここまで

        // Excelファイルを出力
        return $this->outputExcel($spreadsheet);
    }

    /**
     * Get download file name.
     * ファイル名を取得する
     *
     * @return string
     */
    public function getFileName() : string {
        return "test.xlsx";
    }
}
```

### インポート

インポートプラグインは、カスタムデータへのインポート処理を独自に実装することができます。既存システムで出力したファイル等をExmentに取り込む場合は、このプラグインの使用をおすすめします。

#### 作成方法

1. **config.json作成**

```json
{
    "uuid": "b5c0a5d2-2716-4161-98d0-b490c1ebc521",
    "plugin_name": "PluginImportContract",
    "plugin_view_name": "契約データのインポート",
    "description": "契約データを独自ロジックでインポートします。",
    "author":  "Kajitori",
    "version": "1.0.0",
    "plugin_type": "import",
    "target_tables": "contract",
    "label": "契約インポート",
    "icon": "fa-files-o"
}
```

2. **PHPファイル作成**

```php
<?php
namespace App\Plugins\Pluginimportcontract;

use Exceedone\Exment\Services\Plugin\PluginImportBase;
use PhpOffice\PhpSpreadsheet\IOFactory;

class Plugin extends PluginImportBase{
    /**
     * execute
     */
    public function execute() {
        $path = $this->file->getRealPath();

        $reader = $this->createReader();
        $spreadsheet = $reader->load($path);

        // Sheet1のB4セルの内容で契約テーブルを読み込みます
        $sheet = $spreadsheet->getSheetByName('Sheet1');
        $client_name = getCellValue('B4', $sheet, true);
        $client = getModelName('client')::where('value->client_name', $client_name)->first();

        // Sheet1のヘッダ部分に記載された情報で契約データを編集します
        // statusには固定値:1を設定します
        $contract = [
            'value->contract_code' => getCellValue('B3', $sheet, true),
            'value->client' => $client->id,
            'value->status' => '1',
            'value->contract_date' => getCellValue('D4', $sheet, true),
        ];
        // 契約テーブルにレコードを追加します
        $record = getModelName('contract')::create($contract);

        // Sheet1の7行目～15行目に記載された明細情報を元に契約明細データを出力します
        for ($i = 7; $i <= 15; $i++) {
            // A列から製品バージョンコードを取得します
            $product_version_code = getCellValue("A$i", $sheet, true);
            // 製品バージョンコードが設定されていない時は次の行にスキップします
            if (!isset($product_version_code)) break;
            // 製品バージョンコードで、製品バージョンテーブルを読み込みます
            $product_version = getModelName('product_version')
                ::where('value->product_version_code', $product_version_code)->first();
            // 製品バージョンテーブルが取得できなかった場合は次の行にスキップします
            if (!isset($product_version)) continue;
            // 明細行と製品バージョンテーブルから契約明細データを編集します
            $contract_detail = [
                'parent_id' => $record->id,
                'parent_type' => 'contract',
                'value->product_version_id' => $product_version->id,
                'value->fixed_price' => getCellValue("B$i", $sheet, true),
                'value->num' => getCellValue("C$i", $sheet, true),
                'value->zeinuki_price' => getCellValue("D$i", $sheet, true),
            ];
            // 契約明細テーブルにレコードを追加します
            getModelName('contract_detail')::create($contract_detail);
        }

        return true;
    }

    /**
     * create reader
     */
    protected function createReader()
    {
        return IOFactory::createReader('Xlsx');
    }
}
```

### CRUDページ

CRUDページプラグインは、Exmentに新しいCRUD（Create, Read, Update, Delete）機能を持つページを追加します。これにより、Exmentの標準機能では対応できない特殊なデータ管理画面を構築できます。

#### 作成方法

1. **config.json作成**

```json
{
    "plugin_name": "MySQLWorld",
    "plugin_view_name": "MySQL World",
    "description": "MySQLのworldデータベースを表示します。",
    "uuid": "your-unique-uuid-here",
    "author": "(Your Name)",
    "version": "1.0.0",
    "plugin_type": "crud"
}
```

2. **PHPファイル作成**

```php
<?php
namespace App\Plugins\MySQLWorld;

use Exceedone\Exment\Services\Plugin\PluginCrudBase;

class Plugin extends PluginCrudBase
{
    /**
     * 一覧画面の表示
     */
    public function grid()
    {
        // CRUDの一覧表示ロジック
        return $this->pluginView('grid', ['data' => $this->getData()]);
    }

    /**
     * 詳細画面の表示
     */
    public function show($id)
    {
        // CRUDの詳細表示ロジック
        $data = $this->getDataById($id);
        return $this->pluginView('show', ['data' => $data]);
    }

    /**
     * 作成画面の表示
     */
    public function create()
    {
        // CRUDの作成画面ロジック
        return $this->pluginView('create');
    }

    /**
     * 編集画面の表示
     */
    public function edit($id)
    {
        // CRUDの編集画面ロジック
        $data = $this->getDataById($id);
        return $this->pluginView('edit', ['data' => $data]);
    }

    /**
     * データの保存
     */
    public function store()
    {
        // CRUDの保存ロジック
        // バリデーション、データ保存処理
        return redirect()->back()->with('success', 'データが保存されました。');
    }

    /**
     * データの更新
     */
    public function update($id)
    {
        // CRUDの更新ロジック
        // バリデーション、データ更新処理
        return redirect()->back()->with('success', 'データが更新されました。');
    }

    /**
     * データの削除
     */
    public function destroy($id)
    {
        // CRUDの削除ロジック
        // データ削除処理
        return redirect()->back()->with('success', 'データが削除されました。');
    }
}
```

### API

APIプラグインは、Exmentに新しいAPIエンドポイントを追加します。これにより、外部システムとの連携や、カスタムアプリケーションからのデータアクセスが可能になります。

#### 作成方法

1. **config.json作成**

```json
{
    "plugin_name": "PluginDemoAPI",
    "plugin_view_name": "デモAPI",
    "description": "APIのデモプラグインです。",
    "uuid": "your-unique-uuid-here",
    "author": "(Your Name)",
    "version": "1.0.0",
    "plugin_type": "api",
    "route": [
        {
            "uri": "test",
            "method": ["get"],
            "function": "test"
        },
        {
            "uri": "data/{id}",
            "method": ["get"],
            "function": "getData"
        }
    ]
}
```

2. **PHPファイル作成**

```php
<?php
namespace App\Plugins\PluginDemoAPI;

use Exceedone\Exment\Services\Plugin\PluginApiBase;

class Plugin extends PluginApiBase
{
    /**
     * テストAPI
     */
    public function test()
    {
        return response()->json([
            'status' => 'success',
            'message' => 'API test successful',
            'timestamp' => now()
        ]);
    }

    /**
     * データ取得API
     */
    public function getData($id)
    {
        // IDに基づいてデータを取得
        $data = $this->getCustomData($id);
        
        if (!$data) {
            return response()->json([
                'status' => 'error',
                'message' => 'Data not found'
            ], 404);
        }

        return response()->json([
            'status' => 'success',
            'data' => $data
        ]);
    }

    /**
     * カスタムデータ取得メソッド
     */
    private function getCustomData($id)
    {
        // ここにデータ取得ロジックを実装
        // 例: データベースからデータを取得
        return [
            'id' => $id,
            'name' => 'Sample Data',
            'created_at' => now()
        ];
    }
}
```

### バリデーション

バリデーションプラグインは、Exmentのデータ入力時に独自のバリデーションルールを追加します。これにより、業務固有の検証ロジックを実装できます。

#### 作成方法

1. **config.json作成**

```json
{
    "plugin_name": "PluginValidatorTest",
    "plugin_view_name": "バリデーションテスト",
    "description": "カスタムバリデーションのテストプラグインです。",
    "uuid": "your-unique-uuid-here",
    "author": "(Your Name)",
    "version": "1.0.0",
    "plugin_type": "validator",
    "target_tables": "your_table_name"
}
```

2. **PHPファイル作成**

```php
<?php
namespace App\Plugins\PluginValidatorTest;

use Exceedone\Exment\Services\Plugin\PluginValidatorBase;

class Plugin extends PluginValidatorBase
{
    /**
     * バリデーション実行
     */
    public function validate($data, $custom_table, $custom_value = null)
    {
        $errors = [];

        // カスタムバリデーションロジック
        if (isset($data['email']) && !$this->isValidEmail($data['email'])) {
            $errors['email'] = 'メールアドレスの形式が正しくありません。';
        }

        if (isset($data['age']) && $data['age'] < 0) {
            $errors['age'] = '年齢は0以上である必要があります。';
        }

        // 複数フィールドの組み合わせバリデーション
        if (isset($data['start_date']) && isset($data['end_date'])) {
            if (strtotime($data['start_date']) > strtotime($data['end_date'])) {
                $errors['end_date'] = '終了日は開始日より後である必要があります。';
            }
        }

        return $errors;
    }

    /**
     * メールアドレスの独自バリデーション
     */
    private function isValidEmail($email)
    {
        // 独自のメールアドレス検証ロジック
        return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
    }
}
```

### Docurain（PDF出力）

DocurainプラグインはExcelとjsonだけで帳票開発ができるクラウド帳票エンジンDocurainを使用し、PDF出力を行います。

#### Docurainとは

Docurainは、Excelとjsonだけで帳票開発ができるクラウド帳票エンジンです。さまざまなレイアウトの帳票も、Excelファイルのテンプレートから作成することができ、またPDF形式の出力にも対応しています。

#### 実行方法

1. 公式プラグインをダウンロードし、Exmentのプラグインとしてアップロードします。
2. アップロード後、Docurainプラグインの設定画面で以下の内容を入力します：
   - 対象テーブル：Docurainを実行する対象のテーブル
   - トークン：Docurain実行トークン
   - テーブル名と帳票ファイル名一覧：帳票出力を行うテーブル名とテンプレートファイル名、出力する帳票ファイル名、ボタンのラベルをカンマ区切りで入力

#### テンプレートExcelファイル作成

帳票の元となるExcelファイルを作成します。テンプレートファイルのパラメータは、通常のExmentのパラメータ記載方法とは異なり、Docurain専用の記載方法が必要です。

##### パラメータ例

- システム値：`%{system.site_name}`, `%{system.site_name_short}`
- データ：`%{id}`, `%{value.(列名)}`, `%{select_table.(列名).(参照先のテーブルの列名)}`
- 親テーブル：`%{parent.(参照先のテーブルの列名)}`
- 子テーブル：`$ENTITY.children.(子テーブル名)`

## イベント（非推奨）

v3.2.0より、「プラグイン(トリガー)」は非推奨になりました。今後は、「プラグイン(ボタン)」もしくは「プラグイン(イベント)」での実装を行ってください。

Exmentの画面上で特定の操作を行った場合に実行され、値の更新などの処理を行うことができます。もしくは、一覧画面もしくはフォーム画面にボタンを追加し、クリック時に処理を行うことができます。

### トリガーの種類

| 名前 | 種類 | 説明 |
|------|------|------|
| saving | 保存直前 | データの保存直前に、処理が起動します。 |
| saved | 保存後 | データの保存後に、処理が起動します。 |
| grid_menubutton | データ一覧画面のメニューボタン | データ一覧画面の上部にボタンを追加し、クリック時にイベントを発生させます。 |
| form_menubutton_show | データ詳細画面のメニューボタン | データ詳細画面の上部にボタンを追加し、クリック時にイベントを発生させます。 |
| form_menubutton_create | データ新規作成画面のメニューボタン | データ新規作成画面の上部にボタンを追加し、クリック時にイベントを発生させます。 |
| form_menubutton_edit | データ編集画面のメニューボタン | データ編集画面の上部にボタンを追加し、クリック時にイベントを発生させます。 |
