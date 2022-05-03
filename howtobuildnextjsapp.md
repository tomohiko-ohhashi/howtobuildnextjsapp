# Next.js､ Prisma､PostgreSQLでフルスタックアプリを作る


以下を翻訳した文章になります｡

How to Build a Fullstack App with Next.js, Prisma, and PostgreSQL

https://vercel.com/guides/nextjs-prisma-postgres

---

[Prisma](https://www.prisma.io/)はNode.jsとTypeScriptのアプリケーションでデータベースにアクセスする次世代のORMである｡この文章は､次の技術を用いてフルスタックのブログアプリを作るガイドです｡

- [Next.js](https://nextjs.org/) Reactのフレームワーク
- [Next.js API](https://nextjs.org/docs/api-routes/introduction) routes サーバーサイドAPIのルーティング
- [Prisma](https://www.prisma.io/) マイグレーションとデータベースアクセスをするORM
- [PostgreSQL](https://www.postgresql.org/) データベース
- [NextAuth.js](https://next-auth.js.org/) GitHub (OAuth)による認証
- [TypeScript](https://www.typescriptlang.org/) プログラミング言語
- [Vercel](https://vercel.com/) アプリをデプロイ

Next.jsでの[Static-Site Generation(SSG)とSercer-Side Renderring(SSR) *リンク切れ](https://vercel.com/https:/nextjs.org/docs/basic-features/data-fetching)を利用し､Vercelにアプリケーションをデプロイしよう｡

## 事前準備
当ガイドでは以下が必要になります｡
- Node.js
- PostgreSQLデータベース ([無料でHerokuにPostgreSQLをセットアップする方法](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1))
- GitHubアカウント (OAuthアプリを作成するために使用)
- Vercelアカウント (作成したアプリをデプロイするために使用)

## ステップ1: Next.jsプロジェクトを作成する
任意のディレクトリに移動して、ターミナルで次のコマンドを実行し、新しいNext.jsプロジェクトを立ち上げます。:
```
$ npx create-next-app --example https://github.com/prisma/blogr-nextjs-prisma/tree/main blogr-nextjs-prisma
```
<font color="Gray">リポジトリからダウンロードし､スタータープロジェクトを新規フォルダに作成｡</font>

次のコマンドで、作成したディレクトリに移動してアプリを起動することができます。
```
$ cd blogr-nextjs-prisma && npm run dev
```
<font color="Gray">https://localhost:3000 でNext.jsアプリを起動します｡</font>

このような画面を見ることができます｡

<img width="85%" alt="localhost:3000" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/1.png">

<font color="Gray">現在のアプリの状況｡</font>

このアプリは現在、'index.ts' ファイルの 'getStaticProps' から返されるハードコードされたデータを表示しています。後のセクションでは、実際のデータベースからデータが返されるように、これを変更します。

## ステップ2: Prismaをセットアップし､PostgreSQLデータベースにアクセスする
このステップでは､無料でHerokuにPostgreSQLデータベースを作ります｡[こちらのガイド](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1)を見てデータベースを作成してください｡

あるいは､[ローカル](https://www.prisma.io/dataguide/postgresql/setting-up-a-local-postgresql-database)のPostgreSQLデータベースを使用することも可能です｡しかし､後のステップでVercelにデプロイされたときにアクセスできるようになっている必要があります｡

次に､Prismaをセットアップし､PostgreSQLと接続します｡Prisma CLIをnpmでインストールする:
```
$ npm install prisma --save-dev
```
<font color="Gray">Prisma CLIのインストール</font>

さて、Prisma CLIを使用して、以下のコマンドで基本的なPrismaのセットアップをブートストラップすることができます:
```
$ npx prisma init
```
<font color="Gray">あなたのアプリ内にPrismaをinit</font>

これにより､'prisma'という新しいディレクトリ内に､次のファイルが作成されます｡
- 'schema.prisma': データベースのスキーマ情報を含むPrismaの設定ファイル
- '.env': データベース接続URLや環境変数といった情報を定義する[dotenv](https://github.com/motdotla/dotenv)ファイルです｡

'.env' ファイルを開き､ダミーの接続URLを､あなたのPostgreSQLデータベース接続URLに書き換えてください｡たとえば､もしHerokuのデータベースを使うなら､URLは次のようになっているかと思います:

    // .env
    DATABASE_URL="postgresql://giwuzwpdnrgtzv:d003c6a604bb400ea955c3abd8c16cc98f2d909283c322ebd8e9164b33ccdb75@ec2-54-170-123-247.eu-west-1.compute.amazonaws.com:5432/d6ajekcigbuca9"

<font color="Gray">データベース接続URLの例</font>


注: データベースがHerokuでホストされている場合､資格情報を表示し接続URLを[ここ](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1#step-4-access-the-database-credentials-and-connection-url)からコピーできます｡


## ステップ3: Prismaでデータベーススキーマを作成する
このステップでは､Prisma CLIを使用してあなたのデータベースにテーブルを作成します｡

モデル定義を'schema.prisma'に追記し､以下のようになるようにします:

    // schema.prisma
    datasource db {
    provider = "postgresql"
    url      = env("DATABASE_URL")
    }

    generator client {
    provider = "prisma-client-js"
    }

    model Post {
    id        Int     @default(autoincrement()) @id
    title     String
    content   String?
    published Boolean @default(false)
    author    User?   @relation(fields: [authorId], references: [id])
    authorId  Int?
    }

    model User {
    id            Int       @default(autoincrement()) @id
    name          String?
    email         String?   @unique
    createdAt     DateTime  @default(now()) @map(name: "created_at")
    updatedAt     DateTime  @updatedAt @map(name: "updated_at")
    posts         Post[]
    @@map(name: "users")
    }

<font color="Gray">Prisma schema</font>


    注: ときどき `@map` や `@@map` を使って、フィールド名やモデル名をデータベースの異なるカラム名やテーブル名にマッピングすることがあります。これは、NextAuth.js が、データベース内のものを特定の方法で呼び出すための特別な要件を備えているからです。

このPrismaスキーマは2つのモデルを定義し、それぞれが基礎となるデータベースのテーブルにマッピングされます。'User'と'Post'です。2つのモデルの間には、'Post'の'author'フィールドと'User'の'posts'フィールドを介したリレーション（一対多）も存在することに注意してください。

実際にデータベースにテーブルを作るため､Prisma CLIに次のコマンドを実行してください:
```
$ npx prisma db push
```
<font color="Gray">Prisma schemaに基づいてデータベースにテーブルを作る｡</font>

次の出力がされます:

    Environment variables loaded from /Users/nikolasburk/Desktop/nextjs-guide/blogr-starter/.env
    Prisma schema loaded from prisma/schema.prisma

    🚀  Your database is now in sync with your schema. Done in 2.10s
<font color="Gray">データベースにPrisma schemaをpushしたときの出力｡</font>

おめでとう､テーブルが作成されました！Prisma Studioを使ってダミーのデータを作ってください｡次のコマンドを実行します:
```
$ npx prisma studio
```
<font color="Gray">Prisma Studioを開き､データベースをGUIで修正します｡</font>

Prisma Studioのインターフェースを使って､新たに 'User'と'Post'のレコードを作成し､それらを関係フィールドで接続します｡

<img width="85%" alt="Prisma Studio" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/2.png">

<font color="Gray">'User'レコードを新たに作成</font>

<img width="85%" alt="Prisma Studio" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/3.png">

<font color="Gray">新たに'Post'レコードを作成し､'User'レコードに接続する</font>

## ステップ3: Prisma Clientのインストールと生成

Prismaを利用してNext.jsからデータベースにアクセスする前に､あなたのアプリにPrisma Clientをインストールする必要があります｡次のコマンドでnpmを利用してインストールしてください:

```
$ npm install @prisma/client
```
<font color="Gray">Prisma Clientパッケージをインストール</font>

Prisma Clientは独自のスキーマに合わせて作られているため、Prismaのスキーマファイルが変更されるたびに、以下のコマンドを実行して更新する必要があります:
```
$ npx prisma generate
```
<font color="Gray">Prisma Schemaの再生成｡</font>

'PrismaClient'のインスタンスを1つだけ使用し、必要なファイルにインポートすることができます。このインスタンスは、'lib/' ディレクトリ内の 'prisma.ts' ファイルに作成されます。次のコマンドで不足しているディレクトリとファイルを作成します。:
```
$ mkdir lib && touch lib/prisma.ts
```
<font color="Gray">Prismaライブラリのために新規ディレクトリを作成｡</font>

そして､次のコードを 'lib/prisma.ts' に追記してください:
    // lib/prisma.ts
    import { PrismaClient } from '@prisma/client';

    let prisma: PrismaClient;

    if (process.env.NODE_ENV === 'production') {
    prisma = new PrismaClient();
    } else {
    if (!global.prisma) {
        global.prisma = new PrismaClient();
    }
    prisma = global.prisma;
    }

    export default prisma;

<font color="Gray">Prisma Clientへの接続を作成｡</font>

これにより、データベースにアクセスする必要があるときはいつでも、必要なファイルに 'prisma' インスタンスをインポートすることができるようになりました。


## ステップ4: データベースからデータを読み込み､表示するようにviewを変更する

'pages/index.tsx' に実装されているブログ記事フィードと 'pages/p/[id].tsx' に実装されている記事詳細ビューは、現在ハードコードされたデータを返しています。このステップでは、Prisma Clientを使用してデータベースからデータを返すように実装を調整します。

'pages/index.tsx' を開き、既存の 'import' 宣言のすぐ下に以下のコードを追加します。:

    // pages/index.tsx
    import prisma from '../lib/prisma';

<font color="Gray">Prisma Clientにインポートする｡</font>

'prisma' のインスタンスは、データベースのデータを読み書きする際のインターフェイスになります。例えば、 'prisma.user.create()' で新しい 'User' レコードを作成したり、'prisma.post.findMany()' でデータベースから全ての 'Post' レコードを取得したりすることができます。Prisma Client API全体の概要については、[Prisma docs](https://www.prisma.io/docs/concepts/components/prisma-client/crud)を参照してください。

これで、'index.tsx' 内の 'getStaticProps' でハードコードされたフィードオブジェクトを、データベースへの適切な呼び出しに置き換えることができます:

    // index.tsx
    export const getStaticProps: GetStaticProps = async () => {
    const feed = await prisma.post.findMany({
        where: { published: true },
        include: {
        author: {
            select: { name: true },
        },
        },
    });
    return { props: { feed } };
    };

<font color="Gray">データベース内のすべての公開記事を検索します。</font>

Prisma Clientクエリで注意すべき2点:
- 'published' が 'true' の'Post' レコードのみを含むように 'where' フィルタを指定する。
- 'Post' の'author' の 'name' レコードも同様に問い合わせられ、返されるオブジェクトに含まれます。

アプリを実行する前に、'/pages/p/[id].tsx'に移動して、データベースから正しい 'Post' レコードを読み込むように、そこでも実装を調整します。

このページでは、'getStaticProps'（SSG）ではなく、'getServerSideProps'（SSR）を使用しています。これは、データが動的で、URLで要求された 'Post' の 'id' に依存するためです。例えば、ルート '/p/42' のビューは、'id' が '42' である 'Post' を表示します。

前回同様に､まずはじめにページ上でPrisma Clientをインポートする必要があります:

    // pages/p/[id].tsx
    import prisma from '../../lib/prisma';

<font color="Gray">Prisma Clientをインポート</font>

これで、'getServerSideProps' の実装を更新して、データベースから適切なポストを取得し、コンポーネントの 'props' を介してフロントエンドで利用できるようにすることができます:

    // pages/p/[id].tsx
    export const getServerSideProps: GetServerSideProps = async ({ params }) => {
    const post = await prisma.post.findUnique({
        where: {
        id: Number(params?.id) || -1,
        },
        include: {
        author: {
            select: { name: true },
        },
        },
    });
    return {
        props: post,
    };
    };

<font color="Gray">IDをもとに特定の投稿を検索する。</font>

以上です。もし、アプリが起動しなくなった場合は、以下のコマンドで再起動することができます:
```
$ npm run dev
```

<font color="Gray">アプリを[http://localhost:3000](http://localhost:3000)で起動</font>

そうでない場合は、ファイルを保存して、アプリを以下の場所で開いてください。
http://localhost:3000
をブラウザで表示します。Postの記録は以下のように表示されます:
<img width="85%" alt="Prisma Studio" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/4.png">

<font color="Gray">新しく公開された投稿</font>

投稿をクリックすると、その詳細表示に移動することができます。

## ステップ5: NextAuthでGitHubの認証を設定する