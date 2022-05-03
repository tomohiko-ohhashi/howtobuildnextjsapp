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

# 事前準備
当ガイドでは以下が必要になります｡
- Node.js
- PostgreSQLデータベース ([無料でHerokuにPostgreSQLをセットアップする方法](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1))
- GitHubアカウント (OAuthアプリを作成するために使用)
- Vercelアカウント (作成したアプリをデプロイするために使用)

# ステップ1. Next.jsプロジェクトを作成する
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

<img width="100%" alt="localhost:3000" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/1.png">

<font color="Gray">現在のアプリの状況｡</font>

このアプリは現在、'index.ts' ファイルの 'getStaticProps' から返されるハードコードされたデータを表示しています。後のセクションでは、実際のデータベースからデータが返されるように、これを変更します。

# ステップ2. Prismaをセットアップし､PostgreSQLデータベースにアクセスする
このステップでは､無料でHerokuにPostgreSQLデータベースを作ります｡[こちらのガイド](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1)を見てデータベースを作成してください｡

あるいは､[ローカル](https://www.prisma.io/dataguide/postgresql/setting-up-a-local-postgresql-database)のPostgreSQLデータベースを使用することも可能です｡しかし､後のステップでVercelにデプロイされたときにアクセスできるようになっている必要があります｡

次に､Prismaをセットアップし､PostgreSQLと接続します｡Prisma CLIをnpmでインストールする:
```
$ npm install prisma --save-dev
```
<font color="Gray">Prisma CLIのインストール</font>

さて、Prisma CLIを使用して、以下のコマンドで基本的なPrismaを開始することができます:
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


>**注:** データベースがHerokuでホストされている場合､資格情報を表示し接続URLを[ここ](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1#step-4-access-the-database-credentials-and-connection-url)からコピーできます｡


# ステップ3-1. Prismaでデータベーススキーマを作成する
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

>**注:** ときどき `@map` や `@@map` を使って、フィールド名やモデル名をデータベースの異なるカラム名やテーブル名にマッピングすることがあります。これは、NextAuth.js が、データベース内のものを特定の方法で呼び出すための特別な要件を備えているからです。

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

<img width="100%" alt="Prisma Studio" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/2.png">

<font color="Gray">'User'レコードを新たに作成</font>

<img width="100%" alt="Prisma Studio" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/3.png">

<font color="Gray">新たに'Post'レコードを作成し､'User'レコードに接続する</font>

# ステップ3-2. Prisma Clientのインストールと生成

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


# ステップ4. データベースからデータを読み込み､表示するようにviewを変更する

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

<img width="100%" alt="localhost:3000" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/4.png">

<font color="Gray">新しく公開された投稿</font>

投稿をクリックすると、その詳細表示に移動することができます。

# ステップ5-1: NextAuthでGitHubの認証を設定する

このステップでは、アプリにGitHub認証を追加します。これにより、認証されたユーザーがUIから投稿を作成、公開、削除できるようにするなど、アプリにさらに機能を追加していきます。

まずはじめに､あなたのアプリにNextAuth.jsライブラリをインストールしてください:

```
npm install next-auth@4 @next-auth/prisma-adapter
```
<font color="Gray">NextAuthライブラリとNextAuth Prisma Adapterをインストールします。</font>

次に､[NextAuthに必要なテーブル](https://next-auth.js.org/adapters/typeorm/postgres) (*リンク切れ)を追加するために､データベーススキーマを変更します｡

データベーススキーマを変更するため､Prismaスキーマを手動で変更し､'prisma db push' コマンドをもういちど実行します｡そして､ 'schema.prisma' を開き､モデルを次のように修正します:

```
// schema.prisma


model Post {
  id        String  @id @default(cuid())
  title     String
  content   String?
  published Boolean @default(false)
  author    User?   @relation(fields: [authorId], references: [id])
  authorId  String?
}

model Account {
  id                 String  @id @default(cuid())
  userId             String  @map("user_id")
  type               String
  provider           String
  providerAccountId  String  @map("provider_account_id")
  refresh_token      String?
  access_token       String?
  expires_at         Int?
  token_type         String?
  scope              String?
  id_token           String?
  session_state      String?
  oauth_token_secret String?
  oauth_token        String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
  @@map("accounts")
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique @map("session_token")
  userId       String   @map("user_id")
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("sessions")
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime? @map("email_verified")
  image         String?
  createdAt     DateTime  @default(now()) @map(name: "created_at")
  updatedAt     DateTime  @updatedAt @map(name: "updated_at")
  posts         Post[]
  accounts      Account[]
  sessions      Session[]

  @@map(name: "users")
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
  @@map("verificationtokens")
}
```
<font color="Gray">Prismaスキーマを更新｡</font>

これらのモデルをさらに知るためには[NextAuth.jsドキュメント](https://next-auth.js.org/adapters/models)を参照してください｡

これで、データベースに実際にテーブルを作成することで、データベーススキーマを調整することができます。以下のコマンドを実行してください:
```
npx prisma db push
```
<font color="Gray">Prismaスキーマに基づき、データベースのテーブルを更新します。</font>

GitHub認証を使うため､新規に[GitHubでOAuth app](https://docs.github.com/en/developers/apps/building-oauth-apps)を作成する必要があります｡はじめに､[GitHub](https://github.com/)にログインします｡そして､[Settings](https://github.com/settings/profile)を開き､[Developer settings](https://github.com/settings/apps)を開きます｡さらに[OAuth Apps](https://github.com/settings/developers)を開きます｡

<img width="100%" alt="GitHub OAuth Apps" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/5.png">

<font color="Gray">GitHubで新規にOAuth applicationを作成します｡</font>

**Register a new application** (または **New OAuth App)** ボタンをクリックし､登録フォームであなたのアプリの情報を入力します｡**Authorization callback URL**は、Next.js '/api/auth' route: 'http://localhost:3000/api/auth' です。

重要なことは､**Authorization callback URL**の入力欄はAuth0のようなもの (コンマで区切った複数のcallback URL) でなく､ひとつのURLのみ入力可能です｡そのため､後々､本番環境にデプロイしたアプリを作りたい場合､再びGitHub OAuth appを作成する必要があります｡
<img width="100%" alt="GitHub OAuth Apps" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/6.png">

<font color="Gray">Authorization callback URLが正しいか確かめてください｡</font>

**Register application**ボタンをクリックし､**Client ID**と**Client Secret**を新規作成します｡それらをコピーし､rootディレクトリの '.env'ファイルに 'GITHUB_ID' と'GITHUB_SECRET' という環境変数として追記します｡また、 'NEXTAUTH_URL' には、GitHubで設定した**Authorization callback URL**と同じ値 'http://localhost:3000/api/auth' を設定してください。

```
# .env

# GitHub OAuth
GITHUB_ID=6bafeb321963449bdf51
GITHUB_SECRET=509298c32faa283f28679ad6de6f86b2472e1bff
NEXTAUTH_URL=http://localhost:3000/api/auth
```

<font color="Gray">完成した.envファイル｡</font>

また、アプリケーション全体にわたってユーザーの認証状態を持続させる必要があります。アプリケーションのルートファイル '_app.tsx' をすばやく変更し、現在のルートコンポーネントを 'next-auth/react' パッケージの 'SessionProvider' でラップしてください。このファイルを開き、現在の内容を次のコードに置き換えます:

```
// _app.tsx

import { SessionProvider } from 'next-auth/react';
import { AppProps } from 'next/app';

const App = ({ Component, pageProps }: AppProps) => {
  return (
    <SessionProvider session={pageProps.session}>
      <Component {...pageProps} />
    </SessionProvider>
  );
};

export default App;
```

<font color="Gray">NextAuth SessionProviderでラップします｡</font>

# ステップ5-2. ログイン機能の追加

ログインボタンといくつかのUIコンポーネントを 'Header.tsx' ファイルに追加していきます｡ファイルを開き､次のコートを貼り付けてください:

```
// Header.tsx
import React from 'react';
import Link from 'next/link';
import { useRouter } from 'next/router';
import { signOut, useSession } from 'next-auth/react';

const Header: React.FC = () => {
  const router = useRouter();
  const isActive: (pathname: string) => boolean = (pathname) =>
    router.pathname === pathname;

  const { data: session, status } = useSession();

  let left = (
    <div className="left">
      <Link href="/">
        <a className="bold" data-active={isActive('/')}>
          Feed
        </a>
      </Link>
      <style jsx>{`
        .bold {
          font-weight: bold;
        }

        a {
          text-decoration: none;
          color: var(--geist-foreground);
          display: inline-block;
        }

        .left a[data-active='true'] {
          color: gray;
        }

        a + a {
          margin-left: 1rem;
        }
      `}</style>
    </div>
  );

  let right = null;

  if (status === 'loading') {
    left = (
      <div className="left">
        <Link href="/">
          <a className="bold" data-active={isActive('/')}>
            Feed
          </a>
        </Link>
        <style jsx>{`
          .bold {
            font-weight: bold;
          }

          a {
            text-decoration: none;
            color: var(--geist-foreground);
            display: inline-block;
          }

          .left a[data-active='true'] {
            color: gray;
          }

          a + a {
            margin-left: 1rem;
          }
        `}</style>
      </div>
    );
    right = (
      <div className="right">
        <p>Validating session ...</p>
        <style jsx>{`
          .right {
            margin-left: auto;
          }
        `}</style>
      </div>
    );
  }

  if (!session) {
    right = (
      <div className="right">
        <Link href="/api/auth/signin">
          <a data-active={isActive('/signup')}>Log in</a>
        </Link>
        <style jsx>{`
          a {
            text-decoration: none;
            color: var(--geist-foreground);
            display: inline-block;
          }

          a + a {
            margin-left: 1rem;
          }

          .right {
            margin-left: auto;
          }

          .right a {
            border: 1px solid var(--geist-foreground);
            padding: 0.5rem 1rem;
            border-radius: 3px;
          }
        `}</style>
      </div>
    );
  }

  if (session) {
    left = (
      <div className="left">
        <Link href="/">
          <a className="bold" data-active={isActive('/')}>
            Feed
          </a>
        </Link>
        <Link href="/drafts">
          <a data-active={isActive('/drafts')}>My drafts</a>
        </Link>
        <style jsx>{`
          .bold {
            font-weight: bold;
          }

          a {
            text-decoration: none;
            color: var(--geist-foreground);
            display: inline-block;
          }

          .left a[data-active='true'] {
            color: gray;
          }

          a + a {
            margin-left: 1rem;
          }
        `}</style>
      </div>
    );
    right = (
      <div className="right">
        <p>
          {session.user.name} ({session.user.email})
        </p>
        <Link href="/create">
          <button>
            <a>New post</a>
          </button>
        </Link>
        <button onClick={() => signOut()}>
          <a>Log out</a>
        </button>
        <style jsx>{`
          a {
            text-decoration: none;
            color: var(--geist-foreground);
            display: inline-block;
          }

          p {
            display: inline-block;
            font-size: 13px;
            padding-right: 1rem;
          }

          a + a {
            margin-left: 1rem;
          }

          .right {
            margin-left: auto;
          }

          .right a {
            border: 1px solid var(--geist-foreground);
            padding: 0.5rem 1rem;
            border-radius: 3px;
          }

          button {
            border: none;
          }
        `}</style>
      </div>
    );
  }

  return (
    <nav>
      {left}
      {right}
      <style jsx>{`
        nav {
          display: flex;
          padding: 2rem;
          align-items: center;
        }
      `}</style>
    </nav>
  );
};

export default Header;
```

<font color="Gray">Headerを通してユーザーがログインすることを許可します｡</font>

ヘッダーがどのようにレンダリングされるのか、その概要を説明します。
- もし認証されたユーザーがいなかったら､**Log in**ボタンを表示します｡
- もしユーザーが認証されていたら､**My drafts**と**New Post**､**Log out**を表示します｡

すでに 'npm run dev' を実行してアプリを起動することができますが、ログインボタンが表示されるようになっていることがわかると思います。しかし、ログインボタンをクリックすると、 http://localhost:3000/api/auth/signin に移動しますが、Next.js は 404 ページをレンダリングします。

どうしてこうなってしまうかと言うと､[NextAuth.jsは認証のための特別な設定](https://next-auth.js.org/configuration/pages)が必要だからです｡次にこの設定を行わなければなりません｡

'pages/api' ディレクトリに次のように新規ディレクトリと新規ファイルを作成します:

```
mkdir -p pages/api/auth && touch pages/api/auth/[...nextauth].ts
```

<font color="Gray">新規ディレクトリとAPIルートを作成します｡</font>

'pages/api/auth/[...nextauth].ts' というこの新規ファイルは､GitHub OAuth認証と[Prisma Adapter](https://next-auth.js.org/adapters/overview#prisma-adapter)で NextAuth.js を設定するために、次の定型文を追加します:

```
// pages/api/auth/[...nextauth].ts

import { NextApiHandler } from 'next';
import NextAuth from 'next-auth';
import { PrismaAdapter } from '@next-auth/prisma-adapter';
import GitHubProvider from 'next-auth/providers/github';
import prisma from '../../../lib/prisma';

const authHandler: NextApiHandler = (req, res) => NextAuth(req, res, options);
export default authHandler;

const options = {
  providers: [
    GitHubProvider({
      clientId: process.env.GITHUB_ID,
      clientSecret: process.env.GITHUB_SECRET,
    }),
  ],
  adapter: PrismaAdapter(prisma),
  secret: process.env.SECRET,
};
```

<font color="Gray">NextAuthとPrisma Adapterを設定します｡</font>

このコードを一度追加したら､ http://localhost:3000/api/auth/signin に再びアクセスします｡今度は**Sign in with GitHub**ボタンが表示されています｡

 <img width="100%" alt="GitHub OAuth Apps" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/7.png">

 <font color="Gray">NextAuthを使用してGitHubでサインイン｡</font>

クリックすると GitHub に転送され、そこで GitHub の認証情報を使って認証を行うことができます。認証が完了すると、再びアプリに戻ります。

>**注:** もしエラーが出て認証出来ない場合､アプリを停止し､'npm run dev' で再起動してください｡

ページ上部､ヘッダーのレイアウトが変更され､認証されたユーザーのものになります｡

<img width="100%" alt="GitHub OAuth Apps" src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/8.png">

<font color="Gray">ヘッダーにログアウトボタンが表示されています｡</font>

# ステップ6. 新規投稿機能の追加

このステップでは､ユーザーが新規投稿する昨日を実装します｡認証されたユーザーは**New post**ボタンをクリックして投稿することができます｡

ボタンはすでに '/create' にルート設定されていますが､現状では実装されていないため､404に飛ばされてしまいます｡

これを修正するため､pagesディレクトリに 'create.tsx' というファイルを新規作成します｡

```
touch pages/create.tsx
```

<font color="Gray">投稿機能用の新規ファイルを作成します｡</font>

今作成したファイルに以下のコードを追加します｡

```
// pages/create.tsx

import React, { useState } from 'react';
import Layout from '../components/Layout';
import Router from 'next/router';

const Draft: React.FC = () => {
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');

  const submitData = async (e: React.SyntheticEvent) => {
    e.preventDefault();
    // TODO
    // You will implement this next ...
  };

  return (
    <Layout>
      <div>
        <form onSubmit={submitData}>
          <h1>New Draft</h1>
          <input
            autoFocus
            onChange={(e) => setTitle(e.target.value)}
            placeholder="Title"
            type="text"
            value={title}
          />
          <textarea
            cols={50}
            onChange={(e) => setContent(e.target.value)}
            placeholder="Content"
            rows={8}
            value={content}
          />
          <input disabled={!content || !title} type="submit" value="Create" />
          <a className="back" href="#" onClick={() => Router.push('/')}>
            or Cancel
          </a>
        </form>
      </div>
      <style jsx>{`
        .page {
          background: var(--geist-background);
          padding: 3rem;
          display: flex;
          justify-content: center;
          align-items: center;
        }

        input[type='text'],
        textarea {
          width: 100%;
          padding: 0.5rem;
          margin: 0.5rem 0;
          border-radius: 0.25rem;
          border: 0.125rem solid rgba(0, 0, 0, 0.2);
        }

        input[type='submit'] {
          background: #ececec;
          border: 0;
          padding: 1rem 2rem;
        }

        .back {
          margin-left: 1rem;
        }
      `}</style>
    </Layout>
  );
};

export default Draft;
```

<font color="Gray">投稿作成機能のコンポーネントを新規作成</font>

このページは 'Layout' コンポーネントにラップされ､ 'Header' や他のUIコンポーネントも含みます｡

これは、いくつかの入力フィールドを持つ単純なフォームをレンダリングします。投稿されると、(今は空の) 'submitData' 関数が呼び出されます。この関数ではReactコンポーネントからAPIルートにデータを渡す必要があり、それによって新しい投稿データの実際のデータベースへの保存を処理することができます。

投稿機能の関数を実装:
```
// /pages/create.tsx

const submitData = async (e: React.SyntheticEvent) => {
  e.preventDefault();
  try {
    const body = { title, content };
    await fetch('/api/post', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
    await Router.push('/drafts');
  } catch (error) {
    console.error(error);
  }
};
```

<font color="Gray">投稿するために､APIルートを呼び出します｡</font>

このコードでは、'useState' を使ってコンポーネントのstateから抽出した 'title' と 'content' プロパティを使用して、'api/post API' ルートへの HTTP POST リクエストで送信しています。

その後、ユーザーを /drafts ページにリダイレクトし、新しく作成された下書きをすぐに見られるようにします。アプリを実行すると、/createルートは以下のようなUIをレンダリングします:

<img width="100%" alt="Create a new draft." src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/9.png">

<font color="Gray">新規下書きを作成｡</font>

'api/post' も '/drafts' ルートもまだ存在しないため､まだこれでは動きません｡

はじめに､ユーザーが送信したPOSTリクエストをバックエンドが処理できるようにしましょう｡[Next.js API routes](https://nextjs.org/docs/api-routes/introduction)機能のおかげで､この機能を実装するためにNext.jsアプリを離れる必要はありません｡ 'pages/api' ディレクトリにファイルを追加すれば良いのです｡

'post' という新規ディレクトリを作成し､ その中に 'index.ts' ファイルを新規作成してください:
```
mkdir -p pages/api/post && touch pages/api/post/index.ts
```

<font color="Gray">投稿用の新規APIルートの作成｡</font>

>**注:** この時点で、余分なディレクトリと 'index.ts' ファイルでルーティングをする代わりに、 'pages/api/post.ts' というファイルを作成することも可能でした。なぜそのようにしないかというと、後で 'api/post' ルートでHTTP DELETEリクエストのための直接的なルートを追加する必要があるからです。後でリファクタリングを省くために、すでに必要な方法でファイルを構成しているのです。

そして､次のコードを 'pages/api/post/index.ts'に追加してください:

```
// pages/api/post/index.ts

import { getSession } from 'next-auth/react';
import prisma from '../../../lib/prisma';

// POST /api/post
// Required fields in body: title
// Optional fields in body: content
export default async function handle(req, res) {
  const { title, content } = req.body;

  const session = await getSession({ req });
  const result = await prisma.post.create({
    data: {
      title: title,
      content: content,
      author: { connect: { email: session?.user?.email } },
    },
  });
  res.json(result);
}
```

<font color="Gray">Prisma Clientを使用してデータベースを変更するためのAPIルートを更新します。</font>

このコードは '/api/post/' ルートでやってきた任意のリクエストのための*handler*関数を実装します｡この実装は次のことをしています:

1. HTTP POSTリクエストのbodyから 'title' と 'content' を抽出する｡
2. NextAuth.jsの 'getSession' ヘルパー関数によって､認証ユーザーから来たリクエストかどうかチェックします｡
3. Prisma Clientを使い､データベースに新規 'Post' レコードを作成する｡

アプリを開き､実装した機能をテストすることができます｡ログインしたあと､タイトルと内容を入力して送信し､確かめてみてください:

<img width="100%" alt="Testing creating a new post via the API Route." src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/10.png">

<font color="Gray">APIルートを使って投稿し､テストしてください｡</font>

**Create**をクリックし､ 'Post' レコードをデーターベースに追加します｡なお、作成直後にリダイレクトされる '/drafts' のルートはまだ 404 が表示されますが、これはすぐに修正します。ちなみにこの状態でも、'npx prisma studio' でPrisma Studioを再度実行すると、新しいPostレコードがデータベースに追加されていることが確認できます。

# ステップ6. 下書き機能の追加

このステップでは､認証されたユーザーに現在の下書きを見ることができるようなページをアプリに追加します｡

このページは、認証されたユーザーに依存しているため、静的にレンダリングすることはできません。このように、認証されたユーザーに基づいて動的にデータを取得するページは、 'getServerSideProps' によるサーバーサイドレンダリング（SSR）の素晴らしい使用例となります。

まず､ 'pages' ディレクトリに 'drafts.tsx' という新規ファイルを作成します｡
```
touch pages/drafts.tsx
```

<font color="Gray">下書き機能のために新規ページを追加します｡</font>

次に､以下のコードをファイルに追加してください:
```
// pages/drafts.tsx

import React from 'react';
import { GetServerSideProps } from 'next';
import { useSession, getSession } from 'next-auth/react';
import Layout from '../components/Layout';
import Post, { PostProps } from '../components/Post';
import prisma from '../lib/prisma';

export const getServerSideProps: GetServerSideProps = async ({ req, res }) => {
  const session = await getSession({ req });
  if (!session) {
    res.statusCode = 403;
    return { props: { drafts: [] } };
  }

  const drafts = await prisma.post.findMany({
    where: {
      author: { email: session.user.email },
      published: false,
    },
    include: {
      author: {
        select: { name: true },
      },
    },
  });
  return {
    props: { drafts },
  };
};

type Props = {
  drafts: PostProps[];
};

const Drafts: React.FC<Props> = (props) => {
  const { data: session } = useSession();

  if (!session) {
    return (
      <Layout>
        <h1>My Drafts</h1>
        <div>You need to be authenticated to view this page.</div>
      </Layout>
    );
  }

  return (
    <Layout>
      <div className="page">
        <h1>My Drafts</h1>
        <main>
          {props.drafts.map((post) => (
            <div key={post.id} className="post">
              <Post post={post} />
            </div>
          ))}
        </main>
      </div>
      <style jsx>{`
        .post {
          background: var(--geist-background);
          transition: box-shadow 0.1s ease-in;
        }

        .post:hover {
          box-shadow: 1px 1px 3px #aaa;
        }

        .post + .post {
          margin-top: 2rem;
        }
      `}</style>
    </Layout>
  );
};

export default Drafts;
```

<font color="Gray">Draftページを更新し､下書きリストを表示できるようにします｡</font>

このReactコンポーネントでは、認証されたユーザーの「下書き」の一覧をレンダリングしています。Prisma Clientによるデータベースへの問い合わせは 'getServerSideProps' で実行されるため、下書きはサーバーサイドレンダリング時にデータベースから取得されます。そしてそのデータは、Reactコンポーネントの 'props' を介して利用できるようになります。

これでアプリの**My drafts**に移動すると、以前作成した未公開の投稿が表示されます。

<img width="100%" alt="Completed drafts page." src="https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/11.png">

<font color="Gray">下書きページを完成させる｡</font>