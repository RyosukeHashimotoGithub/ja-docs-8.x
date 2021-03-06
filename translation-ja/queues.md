# キュー

- [イントロダクション](#introduction)
    - [接続 Vs. キュー](#connections-vs-queues)
    - [ドライバの注意事項と要件](#driver-prerequisites)
- [ジョブの作成](#creating-jobs)
    - [ジョブクラスの生成](#generating-job-classes)
    - [クラス構成](#class-structure)
    - [ジョブミドルウェア](#job-middleware)
- [ジョブのディスパッチ](#dispatching-jobs)
    - [遅延ディスパッチ](#delayed-dispatching)
    - [同期ディスパッチ](#synchronous-dispatching)
    - [ジョブのチェーン](#job-chaining)
    - [キューと接続のカスタマイズ](#customizing-the-queue-and-connection)
    - [最大試行回数／タイムアウト値の指定](#max-job-attempts-and-timeout)
    - [レート制限](#rate-limiting)
    - [エラー処理](#error-handling)
- [ジョブのバッチ](#job-batching)
    - [Batchableジョブの定義](#defining-batchable-jobs)
    - [バッチのディスパッチ](#dispatching-batches)
    - [バッチへのジョブ追加](#adding-jobs-to-batches)
    - [バッチの調査](#inspecting-batches)
    - [バッチのキャンセル](#cancelling-batches)
    - [バッチの失敗](#batch-failures)
- [クロージャのキュー投入](#queueing-closures)
- [キューワーカの実行](#running-the-queue-worker)
    - [キュープライオリティ](#queue-priorities)
    - [キューワーカとデプロイ](#queue-workers-and-deployment)
    - [ジョブの期限切れとタイムアウト](#job-expirations-and-timeouts)
- [Supervisor設定](#supervisor-configuration)
- [失敗したジョブの処理](#dealing-with-failed-jobs)
    - [ジョブ失敗後のクリーンアップ](#cleaning-up-after-failed-jobs)
    - [ジョブ失敗イベント](#failed-job-events)
    - [失敗したジョブの再試行](#retrying-failed-jobs)
    - [不明なモデルの無視](#ignoring-missing-models)
- [キュー上のジョブのクリア](#clearing-jobs-from-queues)
- [ジョブイベント](#job-events)

<a name="introduction"></a>
## イントロダクション

> {tip} 現在、LaravelはRedisで動作するキューのための美しいダッシュボードと設定システムを備えたHorizonを提供しています。詳細は、[Horizonのドキュメント](/docs/{{version}}/horizon)で確認してください。

Laravelのキューサービスは、Beanstalk、Amazon SQS、Redis、さらにはリレーショナル・データベースなどさまざまなキューバックエンドに対し共通のAPIを提供しています。キューによりメール送信のような時間を費やす処理を遅らせることが可能です。時間のかかるタスクを遅らせることで、よりアプリケーションのリクエストをドラマチックにスピードアップできます。

キューの設定ファイルは`config/queue.php`です。このファイルにはフレームワークに含まれているそれぞれのドライバーへの接続設定が含まれています。それにはデータベース、[Beanstalkd](https://beanstalkd.github.io/)、[Amazon SQS](https://aws.amazon.com/sqs)、[Redis](https://redis.io)、ジョブが即時に実行される同期（ローカル用途）ドライバーが含まれています。 `null`キュードライバはキューされたジョブが実行されないように、破棄します。

<a name="connections-vs-queues"></a>
### 接続 Vs. キュー

Laravelのキューへ取り掛かる前に、「接続」と「キュー」の区別を理解しておくことが重要です。`config/queue.php`設定ファイルの中には、`connections`設定オプションがあります。このオプションはAmazon SQS、Beanstalk、Redisなどのバックエンドサービスへの個々の接続を定義します。しかし、どんな指定されたキュー接続も、複数の「キュー」を持つことができます。「キュー」とはキュー済みのジョブのスタック、もしくは積み重ねのことです。

`queue`接続ファイルの`queue`属性を含んでいる、各接続設定例に注目してください。ジョブがディスパッチされ、指定された接続へ送られた時にのデフォルトキューです。言い換えれば、どのキューへディスパッチするのか明確に定義していないジョブをディスパッチすると、そのジョブは接続設定の`queue`属性で定義したキューへ送られます。

    // このジョブはデフォルトキューへ送られる
    Job::dispatch();

    // このジョブは"emails"キューへ送られる
    Job::dispatch()->onQueue('emails');

あるアプリケーションでは複数のキューへジョブを送る必要はなく、代わりに１つのシンプルなキューが適しているでしょう。しかし、複数のキューへジョブを送ることは優先順位づけしたい、もしくはジョブの処理を分割したいアプリケーションでとくに便利です。Laravelのキューワーカはプライオリティによりどのキューで処理するかを指定できるからです。たとえば、ジョブを`high`キューへ送れば、より高い処理プライオリティのワーカを実行できます。

    php artisan queue:work --queue=high,default

<a name="driver-prerequisites"></a>
### ドライバの注意事項と要件

#### データベース

`database`キュードライバを使用するには、ジョブを記録するためのデータベーステーブルが必要です。このテーブルを作成するマイグレーションは`queue:table` Artisanコマンドにより生成できます。マイグレーションが生成されたら、`migrate`コマンドでデータベースをマイグレートしてください。

    php artisan queue:table

    php artisan migrate

#### Redis

`redis`キュードライバーを使用するには、`config/database.php`設定ファイルでRedisのデータベースを設定する必要があります。

**Redisクラスタ**

Redisキュー接続でRedisクラスタを使用している場合は、キュー名に[キーハッシュタグ](https://redis.io/topics/cluster-spec#keys-hash-tags)を含める必要があります。これはキューに指定した全Redisキーが同じハッシュスロットに確実に置かれるようにするためです。

    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => '{default}',
        'retry_after' => 90,
    ],

**ブロッキング**

Redisキューを使用する場合、ワーカのループの繰り返しとRedisデータベースに対する再ポールの前に、ジョブを実行可能にするまでどの程度待つのかを指定する、`block_for`設定オプションを使うことができます。

新しいジョブを得るため、Redisデータベースに連続してポールしてしまうより、キューの負荷にもとづきより効率的になるよう、この値を調整してください。たとえば、ジョブを実行可能にするまで、ドライバーが５秒間ブロックするように指示するには、値に`5`をセットします。

    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => 'default',
        'retry_after' => 90,
        'block_for' => 5,
    ],

> {note} `block_for`へ`0`を設定するとジョブが利用可能になるまで、キューワーカを無制限にブロックしてしまいます。これはさらに、次のジョブが処理されるまで、`SIGTERM`のようなシグナルが処理されるのも邪魔してしまいます。

#### 他のドライバの要件

以下の依存パッケージがリストしたキュードライバを使用するために必要です。

<div class="content-list" markdown="1">
- Amazon SQS: `aws/aws-sdk-php ~3.0`
- Beanstalkd: `pda/pheanstalk ~4.0`
- Redis: `predis/predis ~1.0`、もしくはphpredis PHP拡張
</div>

<a name="creating-jobs"></a>
## ジョブの作成

<a name="generating-job-classes"></a>
### ジョブクラスの生成

キュー投入可能なアプリケーションの全ジョブは、デフォルトで`app/Jobs`ディレクトリへ保存されます。`app/Jobs`ディレクトリが存在しなくても、`make:job` Artisanコマンドの実行時に生成されます。新しいキュージョブをArtisan CLIで生成できます。

    php artisan make:job ProcessPodcast

非同期で実行するため、ジョブをキューへ投入することをLaravelへ知らせる、`Illuminate\Contracts\Queue\ShouldQueue`インターフェイスが生成されたクラスには実装されます。

> {tip} Job stubs may be customized using [stub publishing](/docs/{{version}}/artisan#stub-customization)

<a name="class-structure"></a>
### クラス構成

ジョブクラスは通常とてもシンプルで、キューによりジョブが処理される時に呼び出される、`handle`メソッドのみで構成されています。手始めに、ジョブクラスのサンプルを見てみましょう。この例は、ポッドキャストの公開サービスを管理し、公開前にアップロードしたポッドキャストファイルを処理する必要があるという仮定です。

    <?php

    namespace App\Jobs;

    use App\Models\Podcast;
    use App\Services\AudioProcessor;
    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Bus\Dispatchable;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Queue\SerializesModels;

    class ProcessPodcast implements ShouldQueue
    {
        use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

        protected $podcast;

        /**
         * 新しいジョブインスタンスの生成
         *
         * @param  Podcast  $podcast
         * @return void
         */
        public function __construct(Podcast $podcast)
        {
            $this->podcast = $podcast;
        }

        /**
         * ジョブの実行
         *
         * @param  AudioProcessor  $processor
         * @return void
         */
        public function handle(AudioProcessor $processor)
        {
            // アップロード済みポッドキャストの処理…
        }
    }

この例中、キュージョブのコンテナーに直接[Eloquentモデル](/docs/{{version}}/eloquent)が渡せることに注目してください。ジョブが使用している`SerializesModels`トレイトによりEloquentモデルとロード済みのリレーションは優雅にシリアライズされ、ジョブが処理される時にアンシリアライズされます。キュー投入されたジョブがコンテナでEloquentモデルを受け取ると、モデルの識別子のみシリアライズされています。ジョブが実際に処理される時、キューシステムは自動的にデータベースから完全なモデルインスタンスとロード済みだったリレーションを再取得します。これらはすべてアプリケーションの完全な透過性のためであり、Eloquentモデルインスタンスをシリアライズするときに発生する問題を防ぐことができます。

`handle`メソッドはキューによりジョブが処理されるときに呼びだされます。ジョブの`handle`メソッドにタイプヒントにより依存を指定できることに注目してください。Laravelの[サービスコンテナ](/docs/{{version}}/container)が自動的に依存を注入します。

もし、どのようにコンテナが依存を`handle`メソッドへ注入するかを完全にコントロールしたい場合は、コンテナの`bindMethod`メソッドを使用します。`bindMethod`メソッドは、ジョブとコンテナを受け取るコールバックを引数にします。コールバックの中で、お望みのまま自由に`handle`メソッドを起動できます。通常は、[サービスプロバイダ](/docs/{{version}}/providers)からこのメソッドを呼び出すべきでしょう。

    use App\Jobs\ProcessPodcast;

    $this->app->bindMethod(ProcessPodcast::class.'@handle', function ($job, $app) {
        return $job->handle($app->make(AudioProcessor::class));
    });

> {note} Rawイメージコンテンツのようなバイナリデータは、キュージョブへ渡す前に、`base64_encode`関数を通してください。そうしないと、そのジョブはキューへ設置する前にJSONへ正しくシリアライズされません。

#### リレーションの処理

ロード済みのリレーションもシリアライズされるため、シリアライズ済みのジョブ文字列は極めて大きくなり得ます。リレーションがシリアライズされるのを防ぐには、プロパティの値を設定するときにモデルの`withoutRelations`メソッドを呼び出してください。このメソッドは、ロード済みのリレーションを外したモデルのインスタンスを返します。

    /**
     * 新しいジョブインスタンスの生成
     *
     * @param  \App\Models\Podcast  $podcast
     * @return void
     */
    public function __construct(Podcast $podcast)
    {
        $this->podcast = $podcast->withoutRelations();
    }

<a name="job-middleware"></a>
### ジョブミドルウェア

ジョブミドルウェアはキュー済みジョブの実行周りのカスタムロジックをラップできるようにし、ジョブ自身の定形コードを減らします。例として、５分毎に１ジョブのみを処理するために、LaravelのRedisレート制限機能を活用する、以下の`handle`メソッドを考えてみましょう。

    /**
     * ジョブの実行
     *
     * @return void
     */
    public function handle()
    {
        Redis::throttle('key')->block(0)->allow(1)->every(5)->then(function () {
            info('Lock obtained...');

            // ジョブの処理…
        }, function () {
            // ロック取得ができない…

            return $this->release(5);
        });
    }

このコードは有効ですが、Redisレート制限ロジックが散らかっているため、`handle`メソッドの構造はうるさくなりました。さらに、レート制限をかけたい他のジョブでもこのレート制限ロジックが重複してしまいます。

handleメソッドの中でレート制限をする代わりに、レート制限を処理するジョブミドルウェアを定義できます。Laravelはジョブミドルウェアの置き場所を決めていないため、アプリケーションのどこにでもジョブミドルウェアを設置できます。この例では、`app/Jobs/Middleware`ディレクトリへミドルウェアを設置しています。

    <?php

    namespace App\Jobs\Middleware;

    use Illuminate\Support\Facades\Redis;

    class RateLimited
    {
        /**
         * キュー済みジョブの処理
         *
         * @param  mixed  $job
         * @param  callable  $next
         * @return mixed
         */
        public function handle($job, $next)
        {
            Redis::throttle('key')
                    ->block(0)->allow(1)->every(5)
                    ->then(function () use ($job, $next) {
                        // ロックを取得した場合の処理…

                        $next($job);
                    }, function () use ($job) {
                        // ロックを取得できなかった処理…

                        $job->release(5);
                    });
        }
    }

ご覧の通り、[ルートミドルウェア](/docs/{{version}}/middleware)と同様に、ジョブミドルウェアも処理するジョブを受け取り、コールバックは処理を続けるため呼び出されます。

ジョブミドルウェアを作成したら、ジョブの`middleware`メソッドから返すことにより、指定します。このメソッドはジョブのスカフォールドを行う`make:job` Artisanコマンドでは作成されないため、ジョブクラスの定義に自身で追加してください。

    use App\Jobs\Middleware\RateLimited;

    /**
     * このジョブが通過する必要のあるミドルウェアの取得
     *
     * @return array
     */
    public function middleware()
    {
        return [new RateLimited];
    }

<a name="dispatching-jobs"></a>
## ジョブのディスパッチ

ジョブクラスを書き上げたら、ジョブクラス自身の`dispatch`メソッドを使い、ディスパッチできます。`dispatch`メソッドへ渡す引数は、ジョブのコンストラクタへ渡されます。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;

    class PodcastController extends Controller
    {
        /**
         * 新ポッドキャストの保存
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // ポッドキャスト作成…

            ProcessPodcast::dispatch($podcast);
        }
    }

条件によりジョブをディスパッチする場合は、`dispatchIf`か`dispatchUnless`を使います。

    ProcessPodcast::dispatchIf($accountActive === true, $podcast);

    ProcessPodcast::dispatchUnless($accountSuspended === false, $podcast);

<a name="delayed-dispatching"></a>
### 遅延ディスパッチ

キュー投入されたジョブの実行を遅らせたい場合は、ジョブのディスパッチ時に`delay`メソッドを使います。例として、ディスパッチ後１０分経つまでは、処理が行われないジョブを指定してみましょう。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;

    class PodcastController extends Controller
    {
        /**
         * 新ポッドキャストの保存
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // ポッドキャスト作成…

            ProcessPodcast::dispatch($podcast)
                    ->delay(now()->addMinutes(10));
        }
    }

> {note} Amazon SQSキューサービスは、最大１５分の遅延時間です。

#### レスポンスをブラウザへ送信後のディスパッチ

別の方法として、ユーザーのブラウザにレスポンスを送り終えるまで、ジョブのディスパッチを遅らせる`dispatchAfterResponse`メソッドがあります。これによりキューされたジョブがまだ実行中であっても、ユーザーはアプリケーションをすぐ使い始めることができます。この方法は通常、メール送信のようなユーザーを数秒待たせるジョブにのみ使うべきでしょう。

    use App\Jobs\SendNotification;

    SendNotification::dispatchAfterResponse();

`dispatch`でクロージャをディスパッチし、`afterResponse`メソッドをチェーンすることで、ブラウザにレスポンスを送り終えたらクロージャを実行することも可能です。

    use App\Mail\WelcomeMessage;
    use Illuminate\Support\Facades\Mail;

    dispatch(function () {
        Mail::to('taylor@laravel.com')->send(new WelcomeMessage);
    })->afterResponse();

<a name="synchronous-dispatching"></a>
### 同期ディスパッチ

ジョブを即時（同期的）にディスパッチしたい場合は、`dispatchSync`メソッドを使用します。このメソッドを使用する場合、そのジョブはキューされずに現在のプロセスで即時実行されます。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;

    class PodcastController extends Controller
    {
        /**
         * 新ポッドキャストの保存
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // ポッドキャスト作成…

            ProcessPodcast::dispatchSync($podcast);
        }
    }

<a name="job-chaining"></a>
### ジョブチェーン

主要なジョブが正しく実行し終えた後に連続して実行する必要がある、キュー投入ジョブのリストをジョブチェーンで指定できます。一連のジョブの内、あるジョブが失敗すると、残りのジョブは実行されません。キュー投入ジョブチェーンを実行するには、`Bus`ファサードが提供する`chain`メソッドを使用します

    use Illuminate\Support\Facades\Bus;

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        new ReleasePodcast,
    ])->dispatch();

ジョブクラスインスタンスのチェーンだけでなく、クロージャもチェーンできます。

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        function () {
            Podcast::update(...);
        },
    ])->dispatch();

> {note} ジョブの削除に`$this->delete()`メソッドを使用しても、チェーンしたジョブの処理を停止できません。チェーンの実行を停止するのは、チェーン中のジョブが失敗した場合のみです。

#### チェーンの接続とキュー

ジョブチェーンで使用するデフォルトの接続とキューを指定したい場合は、`onConnection`と`onQueue`メソッドを使用します。これらのメソッドは、キューされたジョブへ別の接続／キューが明確に指定されていない限り使用される、接続とキューを設定します。

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        new ReleasePodcast,
    ])->onConnection('redis')->onQueue('podcasts')->dispatch();

#### チェーンの失敗

ジョブをチェーンするときに、そのチェーンの中のジョブが失敗した時に実行されるクロージャを`chain`メソッドに指定できます。指定コールバックはそのジョブの失敗を引き起こした例外のインスタンスを引数に取ります。

    use Illuminate\Support\Facades\Bus;
    use Throwable;

    Bus::chain([
        new ProcessPodcast,
        new OptimizePodcast,
        new ReleasePodcast,
    ])->catch(function (Throwable $e) {
        // チェーンの中のジョブが失敗した
    })->dispatch();

<a name="customizing-the-queue-and-connection"></a>
### キューと接続のカスタマイズ

#### 特定キューへのディスパッチ

ジョブを異なるキューへ投入することで「カテゴライズ」できますし、さまざまなキューにいくつのワーカを割り当てるかと言うプライオリティ付けもできます。これはキー設定ファイルで定義した、別々のキュー「接続」へのジョブ投入を意味してはいないことに気をつけてください。一つの接続内の複数のキューを指定する方法です。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;

    class PodcastController extends Controller
    {
        /**
         * 新ポッドキャストの保存
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // ポッドキャスト作成…

            ProcessPodcast::dispatch($podcast)->onQueue('processing');
        }
    }

#### 特定の接続へのディスパッチ

複数のキュー接続を利用するなら、ジョブを投入するキューを指定できます。ジョブをディスパッチする時に、`onConnection`メソッドで接続を指定します。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Jobs\ProcessPodcast;
    use Illuminate\Http\Request;

    class PodcastController extends Controller
    {
        /**
         * 新ポッドキャストの保存
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // ポッドキャスト作成…

            ProcessPodcast::dispatch($podcast)->onConnection('sqs');
        }
    }

ジョブを投入する接続とキューを指定するために、`onConnection`と`onQueue`メソッドをチェーンすることもできます。

    ProcessPodcast::dispatch($podcast)
                  ->onConnection('sqs')
                  ->onQueue('processing');

<a name="max-job-attempts-and-timeout"></a>
### 最大試行回数／タイムアウト値の指定

#### 最大試行回数

ジョブが試行する最大回数を指定するアプローチの一つは、Artisanコマンドラインへ`--tries`スイッチ使う方法です。

    php artisan queue:work --tries=3

しかし、より粒度の高いアプローチは、ジョブクラス自身に最大試行回数を定義する方法です。これはコマンドラインで指定された値より、優先度が高くなっています。

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * 最大試行回数
         *
         * @var int
         */
        public $tries = 5;
    }

<a name="time-based-attempts"></a>
#### 時間ベースの試行

失敗するまでジョブの試行を何度認めるかを定義する代わりに、ジョブのタイムアウト時間を定義することもできます。これにより、指定した時間内で複数回ジョブを試行します。タイムアウト時間を定義するには、ジョブクラスに`retryUntil`メソッドを追加します。

    /**
     * タイムアウトになる時間を決定
     *
     * @return \DateTime
     */
    public function retryUntil()
    {
        return now()->addSeconds(5);
    }

> {tip} キューイベントリスナでも、`retryUntil`メソッドを定義できます。

#### Max例外

ジョブを何度も再試行するように指定している場合、指定した回数の例外が発生したことをきっかけにしてその再試行を失敗として取り扱いたい場合も起きると思います。そうするにはジョブクラスに`maxExceptions`プロパティを定義してください。

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * 最大試行回数
         *
         * @var int
         */
        public $tries = 25;

        /**
         * 失敗と判定するまで許す最大例外数
         *
         * @var int
         */
        public $maxExceptions = 3;

        /**
         * ジョブの実行
         *
         * @return void
         */
        public function handle()
        {
            Redis::throttle('key')->allow(10)->every(60)->then(function () {
                // ロックが取得でき、ポッドキャストの処理を行う…
            }, function () {
                // ロックが取得できなかった
                return $this->release(10);
            });
        }
    }

この例の場合、アプリケーションがRedisのロックを取得できない場合は、そのジョブは１０秒でリリースされます。そして、２５回再試行を継続します。しかし発生した例外を３回処理しなかった場合、ジョブは失敗します。

#### タイムアウト

> {note} ジョブのタイムアウトを利用するには、`pcntl`PHP拡張をインストールする必要があります。

同様に、ジョブの最大実行秒数を指定するために、Artisanコマンドラインに`--timeout`スイッチを指定できます。

    php artisan queue:work --timeout=30

しかしながら、最大実行秒数をジョブクラス自身に定義することもできます。ジョブにタイムアウト時間を指定すると、コマンドラインに指定されたタイムアウトよりも優先されます。

    <?php

    namespace App\Jobs;

    class ProcessPodcast implements ShouldQueue
    {
        /**
         * ジョブがタイムアウトになるまでの秒数
         *
         * @var int
         */
        public $timeout = 120;
    }

ソケットやHTTP接続の送信などのＩＯブロッキングプロセスは、指定したタイムアウトを考慮しません。そのため、こうした機能を使用する場合は、それらのAPIを使用して常にタイムアウトを指定してください。たとえばGuzzleを使用する場合は、常に接続とリクエストのタイムアウト値を指定する必要があります。

<a name="rate-limiting"></a>
### レート制限

> {note} この機能が動作するには、アプリケーションで[Redisサーバ](/docs/{{version}}/redis)が利用できる必要があります。

アプリケーションでRedisを利用しているなら、時間と回数により、キュージョブを制限できます。この機能は、キュージョブがレート制限のあるAPIに関連している場合に役立ちます。

`throttle`メソッドの使用例として、指定したジョブタイプを６０秒毎に１０回だけ実行できるように制限しましょう。ロックできなかった場合、あとで再試行できるように、通常はジョブをキューへ戻す必要があります。

    Redis::throttle('key')->allow(10)->every(60)->then(function () {
        // ジョブのロジック処理…
    }, function () {
        // ロックできなかった場合の処理…

        return $this->release(10);
    });

> {tip} 上記の例で`key`は、レート制限したいジョブのタイプを表す一意の認識文字列です。たとえば、ジョブのクラス名と（そのジョブに含まれているならば）EloquentモデルのIDを元に、制限できます。

> {note} レート制限に引っかかったジョブをキューへ戻す(release)する場合も、ジョブの総試行回数(attempts)は増加します。

もしくは、ジョブを同時に処理するワーカの最大数を指定可能です。これは、一度に一つのジョブが更新すべきリソースを変更するキュージョブを使用する場合に、役立ちます。`funnel`メソッドの使用例として、一度に１ワーカのみにより処理される、特定のタイプのジョブを制限してみましょう。

    Redis::funnel('key')->limit(1)->then(function () {
        // ジョブのロジック処理…
    }, function () {
        // ロックできなかった場合の処理…

        return $this->release(10);
    });

> {tip} レート制限を使用する場合、実行を成功するまでに必要な試行回数を決めるのは、難しくなります。そのため、レート制限は[時間ベースの試行](#time-based-attempts)と組み合わせるのが便利です。

<a name="error-handling"></a>
### エラー処理

ジョブの処理中に例外が投げられると、ジョブは自動的にキューへ戻され、再試行されます。ジョブはアプリケーションが許している最大試行回数に達するまで、連続して実行されます。最大試行回数は`queue:work` Artisanコマンドへ`--tries`スイッチを使い定義されます。もしくは、ジョブクラス自身に最大試行回数を定義することもできます。キューワーカの実行についての情報は、[以降](#running-the-queue-worker)で説明します。

<a name="job-batching"></a>
## ジョブのバッチ

Laravel's job batching feature allows you to easily execute a batch of jobs and then perform some action when the batch of jobs has completed executing. Before getting started, you should create a database migration to build a table that will contain your job batch meta information. This migration may be generated using the `queue:batches-table` Artisan command:

    php artisan queue:batches-table

    php artisan migrate

<a name="defining-batchable-jobs"></a>
### Batchableジョブの定義

To build a batchable job, you should [create a queueable job](#creating-jobs) as normal; however, you should add the `Illuminate\Bus\Batchable` trait to the job class. This trait provides access to a `batch` method which may be used to retrieve the current batch that the job is executing in:

    <?php

    namespace App\Jobs;

    use App\Models\Podcast;
    use App\Services\AudioProcessor;
    use Illuminate\Bus\Batchable;
    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Bus\Dispatchable;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Queue\SerializesModels;

    class ProcessPodcast implements ShouldQueue
    {
        use Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

        /**
         * ジョブの実行
         *
         * @return void
         */
        public function handle()
        {
            if ($this->batch()->cancelled()) {
                // Detected cancelled batch...

                return;
            }

            // Batched job executing...
        }
    }

<a name="dispatching-batches"></a>
### バッチのディスパッチ

バッチジョブをディスパッチするには、`Bus`ファサードの`batch`メソッドを使います。もちろんバッチは完了のコールバックと組み合わせるときに便利です。そのためにバッチの完了コールバックを定義づけるために、`then`、`catch`、`finally`メソッドを使ってください。これらのコールバックはそれぞれ起動時に`Illuminate\Bus\Batch`インスタンスを引数に受けます。

    use App\Jobs\ProcessPodcast;
    use App\Podcast;
    use Illuminate\Bus\Batch;
    use Illuminate\Support\Facades\Bus;
    use Throwable;

    $batch = Bus::batch([
        new ProcessPodcast(Podcast::find(1)),
        new ProcessPodcast(Podcast::find(2)),
        new ProcessPodcast(Podcast::find(3)),
        new ProcessPodcast(Podcast::find(4)),
        new ProcessPodcast(Podcast::find(5)),
    ])->then(function (Batch $batch) {
        // すべてのジョブが成功して完了した
    })->catch(function (Batch $batch, Throwable $e) {
        // バッチジョブの失敗をはじめて検出した
    })->finally(function (Batch $batch) {
        // バッチの実行を終了した
    })->dispatch();

    return $batch->id;

#### バッチの名前付け

Laravel HorizonやLaravel Telescopeなどの一部のツールは、バッチに名前が付けられている場合、ユーザーフレンドリーなバッチのデバッグ情報を提供します。バッチに任意の名前を割り当てるには、バッチを定義するときに `name`メソッドを呼び出します。

    $batch = Bus::batch([
        // ...
    ])->then(function (Batch $batch) {
        // 全ジョブが成功して完了した
    })->name('Process Podcasts')->dispatch();

#### バッチの接続とキュー

If you would like to specify the connection and queue that should be used for the batched jobs, you may use the `onConnection` and `onQueue` methods:バッチのジョブに使用する接続とキューを指定する場合は、`onConnection`と`onQueue`メソッドを使用します。

    $batch = Bus::batch([
        // ...
    ])->then(function (Batch $batch) {
        // 全ジョブが成功して完了した
    })->onConnection('redis')->onQueue('podcasts')->dispatch();

<a name="adding-jobs-to-batches"></a>
### バッチへのジョブ追加

バッチ処理しているジョブの中から追加のジョブをバッチに追加できると便利です。このパターンは数千のジョブをバッチする必要があり、Webリクエストでディスパッチするには長すぎる場合で便利でしょう。このような場合は代わりに初期バッチへ「ローダ」ジョブをディスパッチし、さらなるジョブをバッチに追加してください。

    $batch = Bus::batch([
        new LoadImportBatch,
        new LoadImportBatch,
        new LoadImportBatch,
    ])->then(function (Batch $batch) {
        // 全ジョブが成功して完了した
    })->name('Import Contacts')->dispatch();

この例では、`LoadImportBatch`ジョブをバッチへ追加ジョブを追加するために使用します。そのためにはジョブの中からアクセスできる、バッチインスタンスの`add`メソッドを使います。

    use App\Jobs\ImportContacts;
    use Illuminate\Support\Collection;

    /**
     * ジョブの実行
     *
     * @return void
     */
    public function handle()
    {
        if ($this->batch()->cancelled()) {
            return;
        }

        $this->batch()->add(Collection::times(1000, function () {
            return new ImportContacts;
        }));
    }

> {note} バッチへジョブを追加できるのは、同じバッチに属しているジョブの中からだけです。

<a name="inspecting-batches"></a>
### バッチの調査

バッチ完了コールバックに提供される`Illuminate\Bus\Batch`メソッドには、指定バッチを操作／調査に役立つさまざまなプロパティとメソッドがあります。

    // バッチのUUID
    $batch->id;

    // バッチの名前（付けられている場合）
    $batch->name;

    // バッチに割り付けられているジョブの数
    $batch->totalJobs;

    // キューで処理されていないジョブの数
    $batch->pendingJobs;

    // 失敗したジョブの数
    $batch->failedJobs;

    // これまで処理したジョブの数
    $batch->processedJobs();

    // バッチの完了パーセンテージ(0-100)
    $batch->progress();

    // バッチが完了したかを表す
    $batch->finished();

    // バッチの実行をキャンセル
    $batch->cancel();

    // バッチの実行がキャンセルされたかを表す
    $batch->cancelled();

#### Returning Batches From Routes

`Illuminate\Bus\Batch`インスタンスはすべてJSONシリアライズ可能です。つまり、完了プログレスを含むバッチに関する情報を含んだJSONペイロードを取得するため、アプリケーションのルートから直接返すことができます。IDによりバッチを取得するには、`Bus`ファサードの`findBatch`メソッドを使ってください。

    use Illuminate\Support\Facades\Bus;
    use Illuminate\Support\Facades\Route;

    Route::get('/batch/{batchId}', function (string $batchId) {
        return Bus::findBatch($batchId);
    });

<a name="cancelling-batches"></a>
### バッチのキャンセル

特定のバッチの実行をキャンセルする必要もあるでしょう。`Illuminate\Bus\Batch`インスタンスの`cancel`メソッドを呼び出します。

    /**
     * ジョブの実行
     *
     * @return void
     */
    public function handle()
    {
        if ($this->user->exceedsImportLimit()) {
            return $this->batch()->cancel();
        }

        if ($this->batch()->cancelled()) {
            return;
        }
    }

<a name="batch-failures"></a>
### バッチの失敗

バッチのジョブが失敗するとき、（指定されていれば）`catch`コールバックが起動されます。このコールバックはバッチの中のジョブが失敗したときのみ起動されます。

#### 失敗の許可

バッチの中のジョブが失敗すると、Laravelは自動的にバッチに「キャンセル済み(cancelled)」のマークを付けます。この振る舞いを無効にする、つまり失敗したジョブにより自動的にバッチにキャンセル済みマークを付けなくできます。そのためには、バッチをディスパッチする時に`allowFailures`メソッドを呼び出してください。

    $batch = Bus::batch([
        // ...
    ])->then(function (Batch $batch) {
        // 全ジョブが成功して完了した
    })->allowFailures()->dispatch();

#### 失敗したバッチジョブの再試行

特定のバッチで失敗したジョブをすべて簡単にリトライできるよう、`queue:retry-batch` ArtisanコマンドをLaravelは提供しています。`queue:retry-batch`コマンドは再試行すべき失敗したジョブを含むバッチのUUIDを引数に指定します。

    php artisan queue:retry-batch 32dbc76c-4f82-4749-b610-a639fe0099b5

<a name="queueing-closures"></a>
## クロージャのキュー投入

ジョブクラスをキューへディスパッチする代わりに、クロージャもディスパッチできます。現在のリクエストサイクルの外で実行する必要のあるシンプルなタスクを素早くキュー投入するときに適しています。キューにクロージャをディスパッチする場合、クロージャのコードは暗号化して著名されるため、転送時に変更できません。

    $podcast = App\Podcast::find(1);

    dispatch(function () use ($podcast) {
        $podcast->publish();
    });

`catch`メソッドを使用すると、キューの再試行設定をすべて費やしても、キュー投入されたクロージャが失敗し完了できないと、指定したクロージャが実行されます。

    use Throwable;

    dispatch(function () use ($podcast) {
        $podcast->publish();
    })->catch(function (Throwable $e) {
        // このジョブは失敗した
    });

<a name="running-the-queue-worker"></a>
## キューワーカの実行

Laravelには、キューに投入された新しいジョブを処理する、キューワーカも含まれています。`queue:work` Artisanコマンドを使いワーカを実行できます。`queue:work`コマンドが起動したら、皆さんが停止するか、ターミナルを閉じるまで実行し続けることに注意してください。

    php artisan queue:work

> {tip} バックグランドで`queue:work`プロセスを永続的に実行し続けるには、キューワーカが止まらずに実行し続けていることを確実にするため、[Supervisor](#supervisor-configuration)のようなプロセスモニタを利用する必要があります。

キューワーカは長時間起動するプロセスで、メモリ上にアプリケーション起動時の状態を保存していることを記憶にとどめてください。そのため、開発段階では[キューワーカの再起動](#queue-workers-and-deployment)を確実に実行してください。付け加えて、アプリケーションにより生成、もしくは変更された静的な状態は、ジョブ間で自動的にリセットされないことも覚えておきましょう。

別の方法として、`queue:listen`コマンドを実行することもできます。`queue:listen`コマンドを使えば更新したコードをリロード、もしくはアプリケーションの状態をリセットしたい場合に、手動でワーカをリスタートする必要がなくなります。しかし、このコマンドは`queue:work`ほど効率はよくありません。

    php artisan queue:listen

#### 接続とキューの指定

どのキュー接続をワーカが使用するのかを指定できます。`work`コマンドで指定する接続名は、`config/queue.php`設定ファイルで定義されている接続と対応します。

    php artisan queue:work redis

指定した接続の特定のキューだけを処理するように、さらにキューワーカをカスタマイズすることもできます。たとえば、メールの処理をすべて、`redis`キュー接続の`emails`キューで処理する場合、以下のコマンドでキューの処理だけを行うワーカを起動できます。

    php artisan queue:work redis --queue=emails

#### 特定の数のジョブの処理

`--once`オプションは、ワーカにキュー中のジョブをひとつだけ処理するように指示します。

    php artisan queue:work --once

`--max-jobs`オプションは指定した数のジョブを処理し、終了するようにワーカへ指示します。指定数のジョブを処理してからワーカは自動的に再起動するため、このオプションは[Supervisor](supervisor-configuration)と組み合わせて使用すると便利です。

    php artisan queue:work --max-jobs=1000

#### キューされたすべてのジョブを処理し、終了する

`--stop-when-empty`オプションは、すべてのジョブを処理してから終了するように、ワーカへ指示するために使用します。このオプションは、LaravelキューがDockerコンテナ中で動作していて、キューが空になった後でコンテナをシャットダウンしたい場合に便利です。

    php artisan queue:work --stop-when-empty

#### 指定秒数間、ジョブを処理する

`--max-time`オプションは指定秒数の間、ジョブを処理し、終了するようにワーカへ指示するために使用します。指定時間ジョブを処理したあとで自動的にワーカが再起動するため、このオプションは[Supervisor](supervisor-configuration)と組み合わせて使用すると便利です。

    // １時間ジョブを処理してから終了する
    php artisan queue:work --max-time=3600

#### リソースの考察

デーモンキューワーカは各ジョブを処理する前に、フレームワークを「再起動」しません。そのため、各ジョブが終了したら、大きなリソースを開放してください。たとえば、GDライブラリでイメージ処理を行ったら、終了前に`imagedestroy`により、メモリを開放してください。

<a name="queue-priorities"></a>
### キュープライオリティ

ときどき、キューをどのように処理するかをプライオリティ付けしたいことも起きます。たとえば、`config/queue.php`で`redis`接続のデフォルト`queue`を`low`に設定したとしましょう。しかし、あるジョブを`high`プライオリティでキューへ投入したい場合です。

    dispatch((new Job)->onQueue('high'));

`low`キュー上のジョブの処理が継続される前に、全`high`キュージョブが処理されることを確実にするには、`work`コマンドのキュー名にコンマ区切りのリストで指定してください。

    php artisan queue:work --queue=high,low

<a name="queue-workers-and-deployment"></a>
### キューワーカとデプロイ

キューワーカは長時間起動プロセスであるため、リスタートしない限りコードの変更を反映しません。ですから、キューワーカを使用しているアプリケーションをデプロイする一番シンプルな方法は、デプロイ処理の間、ワーカをリスタートすることです。`queue:restart`コマンドを実行することで、全ワーカを穏やかに再起動できます。

    php artisan queue:restart

このコマンドは存在しているジョブが失われないように、現在のジョブの処理が終了した後に、全キューワーカへ穏やかに「終了する(die)」よう指示します。キューワーカは`queue:restart`コマンドが実行されると、終了するわけですから、キュージョブを自動的に再起動する、Supervisorのようなプロセスマネージャーを実行すべきでしょう。

> {tip} このコマンドはリスタートシグナルを保存するために、[キャッシュ](/docs/{{version}}/cache)を使用します。そのため、この機能を使用する前に、アプリケーションのキャッシュドライバーが、正しく設定されていることを確認してください。

<a name="job-expirations-and-timeouts"></a>
### ジョブの期限切れとタイムアウト

#### ジョブの有効期限

`config/queue.php`設定ファイルの中で、各キュー接続は`retry_after`オプションを定義しています。このオプションは処理中のジョブを再試行するまで、キュー接続を何秒待つかを指定します。たとえば、`retry_after`の値が`90`であれば、そのジョブは９０秒の間に削除されることなく処理され続ければ、キューへ再投入されます。通常、`retry_after`値はジョブが処理を妥当に完了するまでかかるであろう秒数の最大値を指定します。

> {note} `retry_after`を含まない唯一の接続は、Amazon SQSです。SQSはAWSコンソールで管理する、[Default Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html)を元にリトライを行います。

#### ワーカタイムアウト

`queue:work` Artisanコマンドは`--timeout`オプションも提供しています。`--timeout`オプションはLaravelキューマスタプロセスが、ジョブを処理する子のキューワーカをKillするまでどのくらい待つかを指定します。さまざまな理由により、時に子のキュープロセスは「フリーズ」します。`--timeout`オプションは、指定した実行時間を過ぎたフリーズプロセスを取り除きます。

    php artisan queue:work --timeout=60

`retry_after`設定オプションと`--timeout` CLIオプションは異なります。しかし、確実にジョブを失わずに、一度だけ処理を完了できるよう共に働きます。

> {note} `--timeout`値は、最低でも数秒`retry_after`設定値よりも短くしてください。これにより、与えられたジョブを処理するワーカが、ジョブのリトライ前に確実にkillされます。`--timeout`オプションを`retry_after`設定値よりも長くすると、ジョブが２度実行されるでしょう。

#### ワーカスリープ時間

ジョブがキュー上に存在しているとき、ワーカは各ジョブ間にディレイを取らずに実行し続けます。`sleep`オプションは、新しく処理するジョブが存在しない時に、どの程度「スリープ」するかを秒単位で指定します。スリープ中、ワーカは新しいジョブを処理しません。ジョブはワーカが目を覚ました後に処理されます。

    php artisan queue:work --sleep=3

<a name="supervisor-configuration"></a>
## Supervisor設定

#### Supervisorのインストール

SupervisorはLinuxオペレーティングシステムのプロセスモニタで、`queue:work`プロセスが落ちると自動的に起動します。UbuntuにSupervisorをインストールするには、次のコマンドを使ってください。

    sudo apt-get install supervisor

> {tip} Supervisorの設定に圧倒されそうならば、Laravelプロジェクトのために、Supervisorを自動的にインストールし、設定する[Laravel Forge](https://forge.laravel.com)の使用を考慮してください。

#### Supervisorの設定

Supervisorの設定ファイルは、通常`/etc/supervisor/conf.d`ディレクトリに保存します。このディレクトリの中には、Supervisorにどのようにプロセスを監視するのか指示する設定ファイルを好きなだけ設置できます。たとえば、`laravel-worker.conf`ファイルを作成し、`queue:work`プロセスを起動、監視させてみましょう。

    [program:laravel-worker]
    process_name=%(program_name)s_%(process_num)02d
    command=php /home/forge/app.com/artisan queue:work sqs --sleep=3 --tries=3
    autostart=true
    autorestart=true
    user=forge
    numprocs=8
    redirect_stderr=true
    stdout_logfile=/home/forge/app.com/worker.log
    stopwaitsecs=3600

この例の`numprocs`ディレクティブは、Supervisorに全部で８つのqueue:workプロセスを実行・監視し、落ちている時は自動的に再起動するよう指示しています。`command`ディレクティブの`queue:work sqs`の部分を変更し、希望のキュー接続に合わせてください。

> {note} 一番時間がかかるジョブが消費する秒数より大きな値を`stopwaitsecs`へ必ず指定してください。そうしないと、Supervisorは処理が終了する前に、そのジョブをキルしてしまうでしょう。

#### Supervisorの起動

設定ファイルができたら、Supervisorの設定を更新し起動するために以下のコマンドを実行してください。

    sudo supervisorctl reread

    sudo supervisorctl update

    sudo supervisorctl start laravel-worker:*

Supervisorの詳細情報は、[Supervisorドキュメント](http://supervisord.org/index.html)で確認してください。

<a name="dealing-with-failed-jobs"></a>
## 失敗したジョブの処理

時より、キューされたジョブは失敗します。心配ありません。物事は計画通りに進まないものです。Laravelではジョブを再試行する最大回数を指定できます。この回数試行すると、そのジョブは`failed_jobs`データベーステーブルに挿入されます。`failed_jobs`テーブルのマイグレーションを生成するには`queue:failed-table`コマンドを実行してください。

    php artisan queue:failed-table

    php artisan migrate

次に[キューワーカ](#running-the-queue-worker)の実行時、`queue:work`コマンドに`--tries`スイッチを付け、最大試行回数を指定します。`--tries`オプションに値を指定しないと、ジョブは１回のみ試行します。

    php artisan queue:work redis --tries=3

さらに、`--backoff`オプションを使用し、失敗してから再試行するまでに何秒待てばよいかをLaravelへ指定できます。デフォルトでは、時間を置かずに再試行します。

    php artisan queue:work redis --tries=3 --backoff=3

ジョブごとに失敗したジョブの再試行までの遅延を設定したい場合は、キュー投入するジョブクラスで`backoff`プロパティを定義してください。

    /**
     * ジョブを再試行するまでに待つ秒数
     *
     * @var int
     */
    public $backoff = 3;

リトライ時のディレイを決める複雑なロジックが必要になる場合は、キュージョブクラスで、`backoff`メソッドを定義してください。

    /**
    * ジョブを再取得する前に何秒待つか計算する
    *
    * @return int
    */
    public function backoff()
    {
        return 3;
    }

`backoff`メソッドからバックオフ値の配列を返すことで、「指数関数的」なバックオフを簡単に設定できます。この例では、再試行の遅延は、１回目の再試行では１秒、２回目の再試行では５秒、３回目の再試行では１０秒になります。

    /**
    * ジョブを再取得する前に何秒待つか計算する
    *
    * @return array
    */
    public function backoff()
    {
        return [1, 5, 10];
    }

<a name="cleaning-up-after-failed-jobs"></a>
### ジョブ失敗後のクリーンアップ

失敗時にジョブ特定のクリーンアップを実行するため、ジョブクラスで`failed`メソッドを直接定義できます。これはユーザーに警告を送ったり、ジョブの実行アクションを巻き戻すために最適な場所です。`failed`メソッドには、そのジョブを落とすことになった`Throwable`例外が渡されます。

    <?php

    namespace App\Jobs;

    use App\Models\Podcast;
    use App\Services\AudioProcessor;
    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Queue\SerializesModels;
    use Throwable;

    class ProcessPodcast implements ShouldQueue
    {
        use InteractsWithQueue, Queueable, SerializesModels;

        protected $podcast;

        /**
         * 新しいジョブインスタンスの生成
         *
         * @param  \App\Models\Podcast  $podcast
         * @return void
         */
        public function __construct(Podcast $podcast)
        {
            $this->podcast = $podcast;
        }

        /**
         * ジョブの実行
         *
         * @param  \App\Services\AudioProcessor  $processor
         * @return void
         */
        public function handle(AudioProcessor $processor)
        {
            // アップロード済みポッドキャストの処理…
        }

        /**
         * ジョブ失敗の処理
         *
         * @param  \Throwable  $exception
         * @return void
         */
        public function failed(Throwable $exception)
        {
            // 失敗の通知をユーザーへ送るなど…
        }
    }

<a name="failed-job-events"></a>
### ジョブ失敗イベント

ジョブが失敗した時に呼び出されるイベントを登録したい場合、`Queue::failing`メソッドが使えます。このイベントはメールや[Slack](https://www.slack.com)により、チームへ通知する良い機会になります。例として、Laravelに含まれている`AppServiceProvider`で、このイベントのコールバックを付け加えてみましょう。

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Queue;
    use Illuminate\Support\ServiceProvider;
    use Illuminate\Queue\Events\JobFailed;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 全アプリケーションサービスの登録
         *
         * @return void
         */
        public function register()
        {
            //
        }

        /**
         * 全アプリケーションサービスの初期処理
         *
         * @return void
         */
        public function boot()
        {
            Queue::failing(function (JobFailed $event) {
                // $event->connectionName
                // $event->job
                // $event->exception
            });
        }
    }

<a name="retrying-failed-jobs"></a>
### 失敗したジョブの再試行

`failed_jobs`データベーステーブルに挿入された、失敗したジョブを全部確認したい場合は`queue:failed` Artisanコマンドを利用します。

    php artisan queue:failed

`queue:failed`コマンドはジョブID、接続、キュー、失敗した時間、その他の情報をリスト表示します。失敗したジョブをジョブIDで指定すると、リトライ可能です。たとえば、IDが`5`の失敗したジョブを再試行するには、以下のコマンドを実行します。

    php artisan queue:retry 5

必要に応じ、複数のIDやIDの範囲（数値IDを使用時）をコマンドへ指定できます。

    php artisan queue:retry 5 6 7 8 9 10

    php artisan queue:retry --range=5-10

失敗したジョブをすべて再試行するには、IDとして`all`を`queue:retry`コマンドへ指定し、実行してください。

    php artisan queue:retry all

失敗したジョブを削除する場合は、`queue:forget`コマンドを使います。

    php artisan queue:forget 5

失敗したジョブを全部削除するには、`queue:flush`コマンドを使います。

    php artisan queue:flush

<a name="ignoring-missing-models"></a>
### 不明なモデルの無視

Eloquentモデルをジョブで取り扱う場合は自動的にキューへ積む前にシリアライズし、ジョブを処理するときにリストアされます。しかし、ジョブがワーカにより処理されるのを待っている間にモデルが削除されると、そのジョブは`ModelNotFoundException`により失敗します。

利便性のため、ジョブの`deleteWhenMissingModels`プロパティを`true`に指定すれば、モデルが見つからない場合自動的に削除できます。

    /**
     * モデルが存在していない場合に、ジョブを削除する
     *
     * @var bool
     */
    public $deleteWhenMissingModels = true;

<a name="clearing-jobs-from-queues"></a>
## キュー上のジョブのクリア

デフォルト接続のデフォルトキューからすべてのジョブを削除したい場合は、`queue:clear` Artisanコマンドを使います。

    php artisan queue:clear

特定の接続やキューからジョブを削除したい場合は、`connection`引数や`queue`オプションも指定できます。

    php artisan queue:clear redis --queue=emails

> {note} キューからのジョブのクリアは、SQS、Redis、およびデータベースキュードライバーでのみ使用できます。さらに、SQSメッセージの削除プロセスには最大６０秒かかるため、コマンドでキューをクリアしてからSQSキューに送信したジョブが削除されるまで最大６０秒かかる可能性があります。

<a name="job-events"></a>
## ジョブイベント

`Queue`[ファサード](/docs/{{version}}/facades)に`before`と`after`メソッドを使い、キューされたジョブの実行前後に実行する、コールバックを指定できます。これらのコールバックはログを追加したり、ダッシュボードの状態を増加させたりするための機会を与えます。通常、これらのメソッドは[サービスプロバイダ](/docs/{{version}}/providers)から呼び出します。たとえば、Laravelに含まれる`AppServiceProvider`を使っていましょう。

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Queue;
    use Illuminate\Support\ServiceProvider;
    use Illuminate\Queue\Events\JobProcessed;
    use Illuminate\Queue\Events\JobProcessing;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 全アプリケーションサービスの登録
         *
         * @return void
         */
        public function register()
        {
            //
        }

        /**
         * 全アプリケーションサービスの初期処理
         *
         * @return void
         */
        public function boot()
        {
            Queue::before(function (JobProcessing $event) {
                // $event->connectionName
                // $event->job
                // $event->job->payload()
            });

            Queue::after(function (JobProcessed $event) {
                // $event->connectionName
                // $event->job
                // $event->job->payload()
            });
        }
    }

`Queue` [ファサード](/docs/{{version}}/facades)の`looping`メソッドを使用し、ワーカがキューからジョブをフェッチする前に、指定したコールバックを実行できます。たとえば、直前の失敗したジョブの未処理のままのトランザクションをロールバックするクロージャを登録できます。

    Queue::looping(function () {
        while (DB::transactionLevel() > 0) {
            DB::rollBack();
        }
    });
