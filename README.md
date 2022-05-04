# Next.js､ Prisma､PostgreSQLでフルスタックアプリを作る

以下を翻訳した文章になります｡

How to Build a Fullstack App with Next.js, Prisma, and PostgreSQL

written by nikolasburk

[https://vercel.com/guides/nextjs-prisma-postgres](https://vercel.com/guides/nextjs-prisma-postgres)


当記事で完成させたアプリをデプロイしています｡

[https://to-blogr.vercel.app/](https://to-blogr.vercel.app/)

***

Next.js､Prisma､PostgreSQLのフルスタックアプリケーションを作成し､Vercelにデプロイする｡

[Prisma](https://www.prisma.io/)はNode.jsとTypeScriptのアプリケーションでデータベースにアクセスする次世代のORMである｡この文章は､次の技術を用いてフルスタックのブログアプリを作るガイドです｡

- [Next.js](https://nextjs.org/) Reactのフレームワーク
- [Next.js API routes](https://nextjs.org/docs/api-routes/introduction) サーバーサイドAPIのルーティング
- [Prisma](https://www.prisma.io/) マイグレーションとデータベースアクセスをするORM
- [PostgreSQL](https://www.postgresql.org/) リレーショナルデータベース
- [NextAuth.js](https://next-auth.js.org/) GitHub (OAuth)による認証
- [TypeScript](https://www.typescriptlang.org/) プログラミング言語
- [Vercel](https://vercel.com/) Next.jsプロジェクトを簡単にデプロイできるサービス

Next.jsでの[Static-Site Generation(SSG)とServer-Side Rendering(SSR) (*リンク切れ) ](https://vercel.com/https:/nextjs.org/docs/basic-features/data-fetching)を利用し､Vercelにアプリケーションをデプロイしよう｡

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

>リポジトリからダウンロードし､スタータープロジェクトを新規フォルダに作成します｡

次のコマンドで、作成したディレクトリに移動してアプリを起動することができます。

```
$ cd blogr-nextjs-prisma && npm run dev
```

>https://localhost:3000 でNext.jsアプリを起動します｡

このような画面を見ることができます｡

![Current state of the application.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/1.png)

>現在のアプリの状況です｡

このアプリは現在、'index.ts' ファイルの 'getStaticProps' から返されるハードコードされたデータを表示しています。後のセクションでは、実際のデータベースからデータが返されるように、これを変更します。

# ステップ2. Prismaをセットアップし､PostgreSQLデータベースにアクセスする

このステップでは､無料でHerokuにPostgreSQLデータベースを作ります｡[こちらのガイド](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1)を見てデータベースを作成してください｡

あるいは､[ローカル](https://www.prisma.io/dataguide/postgresql/setting-up-a-local-postgresql-database)のPostgreSQLデータベースを使用することも可能です｡しかし､後のステップでVercelにデプロイされたときにアクセスできるようになっている必要があります｡

次に､Prismaをセットアップし､PostgreSQLと接続します｡Prisma CLIをnpmでインストールする:

```
$ npm install prisma --save-dev
```

>Prisma CLIをインストールします｡

さて、Prisma CLIを使用して、以下のコマンドで基本的なPrismaを開始することができます:

```
$ npx prisma init
```

>あなたのアプリ内にPrismaを開始します｡

これにより､'prisma'という新しいディレクトリ内に､次のファイルが作成されます｡
- 'schema.prisma': データベースのスキーマ情報を含むPrismaの設定ファイル

また､rootディレクトリに次のファイルも作成されます｡
- '.env': データベース接続URLや環境変数といった情報を定義する[dotenv](https://github.com/motdotla/dotenv)ファイルです｡

'.env' ファイルを開き､ダミーの接続URLを､あなたのPostgreSQLデータベース接続URLに書き換えてください｡たとえば､もしHerokuのデータベースを使うなら､URLは次のようになっているかと思います:

```
// .env
DATABASE_URL="postgresql://giwuzwpdnrgtzv:d003c6a604bb400ea955c3abd8c16cc98f2d909283c322ebd8e9164b33ccdb75@ec2-54-170-123-247.eu-west-1.compute.amazonaws.com:5432/d6ajekcigbuca9"
```

>データベース接続URLの例です｡


>**注:** データベースがHerokuでホストされている場合､資格情報を表示し接続URLを[ここ](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1#step-4-access-the-database-credentials-and-connection-url)からコピーできます｡


# ステップ3-1. Prismaでデータベーススキーマを作成する
このステップでは､Prisma CLIを使用してあなたのデータベースにテーブルを作成します｡

モデル定義を'schema.prisma'に追記し､以下のようになるようにします:

```
// schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Post {
  id        String     @default(cuid()) @id
  title     String
  content   String?
  published Boolean @default(false)
  author    User?   @relation(fields: [authorId], references: [id])
  authorId  String?
}

model User {
  id            String       @default(cuid()) @id
  name          String?
  email         String?   @unique
  createdAt     DateTime  @default(now()) @map(name: "created_at")
  updatedAt     DateTime  @updatedAt @map(name: "updated_at")
  posts         Post[]
  @@map(name: "users")
}
```

>Prismaスキーマです｡

>**注:** ときどき `@map` や `@@map` を使って、フィールド名やモデル名をデータベースの異なるカラム名やテーブル名にマッピングすることがあります。これは、NextAuth.js が、データベース内のものを特定の方法で呼び出すための特別な要件を備えているからです。

このPrismaスキーマは2つのモデルを定義し、それぞれが基礎となるデータベースのテーブルにマッピングされます。'User'と'Post'です。2つのモデルの間には、'Post'の'author'フィールドと'User'の'posts'フィールドを介したリレーション（一対多）も存在することに注意してください。

実際にデータベースにテーブルを作るため､Prisma CLIに次のコマンドを実行してください:

```
$ npx prisma db push
```

>Prisma schemaに基づいてデータベースにテーブルを作ります｡

次の出力がされます:

```
Environment variables loaded from /Users/nikolasburk/Desktop/nextjs-guide/blogr-starter/.env
Prisma schema loaded from prisma/schema.prisma

🚀  Your database is now in sync with your schema. Done in 2.10s
```

>データベースにPrisma schemaをpushしたときの出力です｡

おめでとう､テーブルが作成されました！Prisma Studioを使ってダミーのデータを作ってください｡次のコマンドを実行します:

```
$ npx prisma studio
```

>Prisma Studioを開き､データベースをGUIで修正します｡

Prisma Studioのインターフェースを使って､新たに 'User'と'Post'のレコードを作成し､それらを関係フィールドで接続します｡

![Create a new `User` record](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/2.png)

>'User'レコードを新たに作成します｡

![Create a new `Post` record and connect it to the `User` record](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/3.png)

>新たに 'Post' レコードを作成し､ 'User' レコードに紐づけます｡

# ステップ3-2. Prisma Clientのインストールと生成

Prismaを利用してNext.jsからデータベースにアクセスする前に､あなたのアプリにPrisma Clientをインストールする必要があります｡次のコマンドでnpmを利用してインストールしてください:

```
$ npm install @prisma/client
```

>Prisma Clientパッケージをインストールします｡

Prisma Clientは独自のスキーマに合わせて作られているため、Prismaのスキーマファイルが変更されるたびに、以下のコマンドを実行して更新する必要があります:

```
$ npx prisma generate
```

>Prisma Schemaを再生成します｡

'PrismaClient'のインスタンスを1つだけ使用し、必要なファイルにインポートすることができます。このインスタンスは、'lib/' ディレクトリ内の 'prisma.ts' ファイルに作成されます。次のコマンドで不足しているディレクトリとファイルを作成します。:

```
$ mkdir lib && touch lib/prisma.ts
```

>Prismaライブラリのために新規ディレクトリを作成します｡

そして､次のコードを 'lib/prisma.ts' に追記してください:

```ts
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
```

>Prisma Clientへの接続を作成します｡

これにより、データベースにアクセスする必要があるときはいつでも、必要なファイルに 'prisma' インスタンスをインポートすることができるようになりました。


# ステップ4. データベースからデータを読み込み､表示するようにviewを変更する

'pages/index.tsx' に実装されているブログ記事フィードと 'pages/p/[id].tsx' に実装されている記事詳細ビューは、現在ハードコードされたデータを返しています。このステップでは、Prisma Clientを使用してデータベースからデータを返すように実装を調整します。

'pages/index.tsx' を開き、既存の 'import' 宣言のすぐ下に以下のコードを追加します:

```tsx
// pages/index.tsx
import prisma from '../lib/prisma';
```

>Prisma Clientにインポートします｡

'prisma' のインスタンスは、データベースのデータを読み書きする際のインターフェイスになります。例えば、 'prisma.user.create()' で新しい 'User' レコードを作成したり、'prisma.post.findMany()' でデータベースから全ての 'Post' レコードを取得したりすることができます。Prisma Client API全体の概要については、[Prisma docs](https://www.prisma.io/docs/concepts/components/prisma-client/crud)を参照してください。

これで、'index.tsx' 内の 'getStaticProps' でハードコードされたフィードオブジェクトを、データベースへの適切な呼び出しに置き換えることができます:

```tsx
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
```

>データベース内のすべての公開記事を検索します。

Prisma Clientクエリで注意すべき2点:
- 'published' が 'true' の'Post' レコードのみを含むように 'where' フィルタを指定する。
- 'Post' の'author' の 'name' レコードも同様に問い合わせられ、返されるオブジェクトに含まれます。

アプリを実行する前に、'/pages/p/[id].tsx'に移動して、データベースから正しい 'Post' レコードを読み込むように、そこでも実装を調整します。

このページでは、'getStaticProps'（SSG）ではなく、'getServerSideProps'（SSR）を使用しています。これは、データが動的で、URLで要求された 'Post' の 'id' に依存するためです。例えば、ルート '/p/42' のビューは、'id' が '42' である 'Post' を表示します。

前回同様に､まずはじめにページ上でPrisma Clientをインポートする必要があります:

```tsx
// pages/p/[id].tsx
import prisma from '../../lib/prisma';
```

>Prisma Clientをインポートします｡

これで、'getServerSideProps' の実装を更新して、データベースから適切なポストを取得し、コンポーネントの 'props' を介してフロントエンドで利用できるようにすることができます:

```tsx
// pages/p/[id].tsx
export const getServerSideProps: GetServerSideProps = async ({ params }) => {
  const post = await prisma.post.findUnique({
    where: {
      id: String(params?.id),
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
```

>IDをもとに特定の投稿を検索します。

以上です。もし、アプリが起動しなくなった場合は、以下のコマンドで再起動することができます:
```
$ npm run dev
```

>アプリを[http://localhost:3000](http://localhost:3000)で起動します｡

そうでない場合は、ファイルを保存して、アプリを以下の場所で開いてください。
http://localhost:3000
をブラウザで表示します。Postの記録は以下のように表示されます:

![Your newly published post.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/4.png)

>新しく公開された投稿です｡

投稿をクリックすると、その詳細表示に移動することができます。

# ステップ5-1: NextAuthでGitHubの認証を設定する

このステップでは、アプリにGitHub認証を追加します。これにより、認証されたユーザーがUIから投稿を作成、公開、削除できるようにするなど、アプリにさらに機能を追加していきます。

まずはじめに､あなたのアプリにNextAuth.jsライブラリをインストールしてください:

```
$ npm install next-auth@4 @next-auth/prisma-adapter
```

>NextAuthライブラリとNextAuth Prisma Adapterをインストールします。

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

>Prismaスキーマを修正します｡

これらのモデルをさらに知るためには[NextAuth.jsドキュメント](https://next-auth.js.org/adapters/models)を参照してください｡

これで、データベースに実際にテーブルを作成することで、データベーススキーマを調整することができます。以下のコマンドを実行してください:

```
$ npx prisma db push
```

>Prismaスキーマに基づき、データベースのテーブルを更新します。

GitHub認証を使うため､新規に[GitHubでOAuth app](https://docs.github.com/en/developers/apps/building-oauth-apps)を作成する必要があります｡はじめに､[GitHub](https://github.com/)にログインします｡そして､[Settings](https://github.com/settings/profile)を開き､[Developer settings](https://github.com/settings/apps)を開きます｡さらに[OAuth Apps](https://github.com/settings/developers)を開きます｡

![Create a new OAuth application inside GitHub.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/5.png)

>GitHubで新規にOAuth applicationを作成します｡

**Register a new application** (または **New OAuth App)** ボタンをクリックし､登録フォームであなたのアプリの情報を入力します｡**Authorization callback URL**は、Next.js '/api/auth' route: 'http://localhost:3000/api/auth' です。

重要なことは､**Authorization callback URL**の入力欄はAuth0のようなもの (コンマで区切った複数のcallback URL) でなく､ひとつのURLのみ入力可能です｡そのため､後々､本番環境にデプロイしたアプリを作りたい場合､再びGitHub OAuth appを作成する必要があります｡
![Ensure your Authorization callback URL is correct.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/6.png)

>Authorization callback URLが正しいか確かめてください｡

**Register application**ボタンをクリックし､**Client ID**と**Client Secret**を新規作成します｡それらをコピーし､rootディレクトリの '.env'ファイルに 'GITHUB_ID' と'GITHUB_SECRET' という環境変数として追記します｡また、 'NEXTAUTH_URL' には、GitHubで設定した**Authorization callback URL**と同じ値 'http://localhost:3000/api/auth' を設定してください。

```
# .env

# GitHub OAuth
GITHUB_ID=6bafeb321963449bdf51
GITHUB_SECRET=509298c32faa283f28679ad6de6f86b2472e1bff
NEXTAUTH_URL=http://localhost:3000/api/auth
```

>完成した.envファイルです｡

また、アプリケーション全体にわたってユーザーの認証状態を持続させる必要があります。アプリケーションのルートファイル '_app.tsx' をすばやく変更し、現在のルートコンポーネントを 'next-auth/react' パッケージの 'SessionProvider' でラップしてください。このファイルを開き、現在の内容を次のコードに置き換えます:

```tsx
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

>NextAuth SessionProviderでラップします｡

# ステップ5-2. ログイン機能の追加

ログインボタンといくつかのUIコンポーネントを 'Header.tsx' ファイルに追加していきます｡ファイルを開き､次のコードを貼り付けてください:

```tsx
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

>Headerを通してユーザーがログインすることを許可します｡

ヘッダーがどのようにレンダリングされるのか、その概要を説明します。
- もし認証されたユーザーがいなかったら､**Log in**ボタンを表示します｡
- もしユーザーが認証されていたら､**My drafts**と**New Post**､**Log out**を表示します｡

すでに 'npm run dev' を実行してアプリを起動することができますが、ログインボタンが表示されるようになっていることがわかると思います。しかし、ログインボタンをクリックすると、 http://localhost:3000/api/auth/signin に移動しますが、Next.js は 404 ページをレンダリングします。

どうしてこうなってしまうかと言うと､[NextAuth.jsは認証のための特別な設定](https://next-auth.js.org/configuration/pages)が必要だからです｡次にこの設定を行わなければなりません｡

'pages/api' ディレクトリに次のように新規ディレクトリと新規ファイルを作成します:

```
$ mkdir -p pages/api/auth && touch pages/api/auth/[...nextauth].ts
```

>新規ディレクトリとAPIルートを作成します｡

'pages/api/auth/[...nextauth].ts' というこの新規ファイルは､GitHub OAuth認証と[Prisma Adapter](https://next-auth.js.org/adapters/overview#prisma-adapter)で NextAuth.js を設定するために、次の定型文を追加します:

```ts
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

>NextAuthとPrisma Adapterを設定します｡

このコードを一度追加したら､ http://localhost:3000/api/auth/signin に再びアクセスします｡今度は**Sign in with GitHub**ボタンが表示されています｡

 ![Sign in with GitHub using NextAuth.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/7.png)

>NextAuthを使用してGitHubでサインイン｡

クリックすると GitHub に転送され、そこで GitHub の認証情報を使って認証を行うことができます。認証が完了すると、再びアプリに戻ります。

>**注:** もしエラーが出て認証出来ない場合､アプリを停止し､'npm run dev' で再起動してください｡

ページ上部､ヘッダーのレイアウトが変更され､認証されたユーザーのものになります｡

![The Header displaying a log out button.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/8.png)

>ヘッダーにログアウトボタンが表示されています｡

# ステップ6-1. 新規投稿機能の追加

このステップでは､ユーザーが新規投稿する機能を実装します｡認証されたユーザーは**New post**ボタンをクリックして投稿することができます｡

ボタンはすでに '/create' にルート設定されていますが､現状では実装されていないため､404に飛ばされてしまいます｡

これを修正するため､pagesディレクトリに 'create.tsx' というファイルを新規作成します｡

```
$ touch pages/create.tsx
```

>投稿機能用の新規ファイルを作成します｡

今作成したファイルに以下のコードを追加します｡

```tsx
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

>投稿作成機能のコンポーネントを新規作成します｡

このページは 'Layout' コンポーネントにラップされ､ 'Header' や他のUIコンポーネントも含みます｡

これは、いくつかの入力フィールドを持つ単純なフォームをレンダリングします。投稿されると、(今は空の) 'submitData' 関数が呼び出されます。この関数ではReactコンポーネントからAPIルートにデータを渡す必要があり、それによって新しい投稿データの実際のデータベースへの保存を処理することができます。

投稿機能の関数を実装:

```tsx
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

>投稿するために､APIルートを呼び出します｡

このコードでは、'useState' を使ってコンポーネントのstateから抽出した 'title' と 'content' プロパティを使用して、'api/post API' ルートへの HTTP POST リクエストで送信しています。

その後、ユーザーを /drafts ページにリダイレクトし、新しく作成された下書きをすぐに見られるようにします。アプリを実行すると、/createルートは以下のようなUIをレンダリングします:

![Create a new draft.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/9.png)

>新規下書きを作成します｡

'api/post' も '/drafts' ルートもまだ存在しないため､まだこれでは動きません｡

はじめに､ユーザーが送信したPOSTリクエストをバックエンドが処理できるようにしましょう｡[Next.js API routes](https://nextjs.org/docs/api-routes/introduction)機能のおかげで､この機能を実装するためにNext.jsアプリを離れる必要はありません｡ 'pages/api' ディレクトリにファイルを追加すれば良いのです｡

'post' という新規ディレクトリを作成し､ その中に 'index.ts' ファイルを新規作成してください:

```
$ mkdir -p pages/api/post && touch pages/api/post/index.ts
```

>投稿用の新規APIルートの作成します｡

>**注:** この時点で、余分なディレクトリと 'index.ts' ファイルでルーティングをする代わりに、 'pages/api/post.ts' というファイルを作成することも可能でした。なぜそのようにしないかというと、後で 'api/post' ルートでHTTP DELETEリクエストのための直接的なルートを追加する必要があるからです。後でリファクタリングを省くために、すでに必要な方法でファイルを構成しているのです。

そして､次のコードを 'pages/api/post/index.ts'に追加してください:

```ts
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

>Prisma Clientを使用してデータベースを変更するためのAPIルートを追加します。

このコードは '/api/post/' ルートでやってきた任意のリクエストのための*handler*関数を実装します｡この実装は次のことをしています:

1. HTTP POSTリクエストのbodyから 'title' と 'content' を抽出する｡
2. NextAuth.jsの 'getSession' ヘルパー関数によって､認証ユーザーから来たリクエストかどうかチェックします｡
3. Prisma Clientを使い､データベースに新規 'Post' レコードを作成する｡

アプリを開き､実装した機能をテストすることができます｡ログインしたあと､タイトルと内容を入力して送信し､確かめてみてください:

![Testing creating a new post via the API Route.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/10.png)

>APIルートを使って投稿し､テストしてください｡

**Create**をクリックし､ 'Post' レコードをデーターベースに追加します｡なお、作成直後にリダイレクトされる '/drafts' のルートはまだ 404 が表示されますが、これはすぐに修正します。ちなみにこの状態でも、'npx prisma studio' でPrisma Studioを再度実行すると、新しいPostレコードがデータベースに追加されていることが確認できます。

# ステップ6-2. 下書き機能の追加

このステップでは､認証されたユーザーに現在の下書きを見ることができるようなページをアプリに追加します｡

このページは、認証されたユーザーに依存しているため、静的にレンダリングすることはできません。このように、認証されたユーザーに基づいて動的にデータを取得するページは、 'getServerSideProps' によるサーバーサイドレンダリング（SSR）の素晴らしい使用例となります。

まず､ 'pages' ディレクトリに 'drafts.tsx' という新規ファイルを作成します｡

```
$ touch pages/drafts.tsx
```

>下書き機能のために新規ページを追加します｡

次に､以下のコードをファイルに追加してください:

```tsx
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

>下書きページを更新し､下書きリストを表示できるようにします｡

このReactコンポーネントでは、認証されたユーザーの「下書き」の一覧をレンダリングしています。Prisma Clientによるデータベースへの問い合わせは 'getServerSideProps' で実行されるため、下書きはサーバーサイドレンダリング時にデータベースから取得されます。そしてそのデータは、Reactコンポーネントの 'props' を介して利用できるようになります。

これでアプリの**My drafts**に移動すると、以前作成した未公開の投稿が表示されます。

![Completed drafts page.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/11.png)

>下書きページを完成させます｡

# ステップ7. 公開機能の追加

下書きを投稿公開画面に移動させるため､"publish (公開)" する機能が必要になります｡それはすなわち､データベースの 'Post'レコードの 'published' フィールドを 'true' に更新することです｡この機能は､現在 'pages/p/[id].tsx' にある投稿詳細画面に実装していきます｡

公開はHTTP PUTリクエストを使い､ "Next.js backend" の 'api/publish' ルートに送られます｡まずはルートを実装しましょう｡

'pages/api' ディレクトリに 'publish' ディレクトリを新規作成しましょう｡そして､ '[id].ts' というファイルをその中に新規作成します｡

```
$ mkdir -p pages/api/publish && touch pages/api/publish/[id].ts
```

>投稿を公開する新規APIルートを作成します｡

そして､以下のコードを作成したファイルに追記してください:

```ts
// pages/api/publish/[id].ts

import prisma from '../../../lib/prisma';

// PUT /api/publish/:id
export default async function handle(req, res) {
  const postId = req.query.id;
  const post = await prisma.post.update({
    where: { id: postId },
    data: { published: true },
  });
  res.json(post);
}
```

>Prisma Clientを使用してデータベースを変更するためのAPIルートを更新します。

URLから 'Post' のIDを取得し、Prisma Clientの 'update' メソッドで 'Post' レコードの 'published' フィールドを 'true' に更新するAPIルートハンドラーの実装です。

つぎに､フロントエンドの 'pages/p/[id].tsx' ファイルに公開機能を実装します｡このファイルを開き､内容を以下のように書き換えてください:

```tsx
// pages/p/[id].tsx

import React from 'react';
import { GetServerSideProps } from 'next';
import ReactMarkdown from 'react-markdown';
import Router from 'next/router';
import Layout from '../../components/Layout';
import { PostProps } from '../../components/Post';
import { useSession } from 'next-auth/react';
import prisma from '../../lib/prisma';

export const getServerSideProps: GetServerSideProps = async ({ params }) => {
  const post = await prisma.post.findUnique({
    where: {
      id: String(params?.id),
    },
    include: {
      author: {
        select: { name: true, email: true },
      },
    },
  });
  return {
    props: post,
  };
};

async function publishPost(id: string): Promise<void> {
  await fetch(`/api/publish/${id}`, {
    method: 'PUT',
  });
  await Router.push('/');
}

const Post: React.FC<PostProps> = (props) => {
  const { data: session, status } = useSession();
  if (status === 'loading') {
    return <div>Authenticating ...</div>;
  }
  const userHasValidSession = Boolean(session);
  const postBelongsToUser = session?.user?.email === props.author?.email;
  let title = props.title;
  if (!props.published) {
    title = `${title} (Draft)`;
  }

  return (
    <Layout>
      <div>
        <h2>{title}</h2>
        <p>By {props?.author?.name || 'Unknown author'}</p>
        <ReactMarkdown children={props.content} />
        {!props.published && userHasValidSession && postBelongsToUser && (
          <button onClick={() => publishPost(props.id)}>Publish</button>
        )}
      </div>
      <style jsx>{`
        .page {
          background: var(--geist-background);
          padding: 2rem;
        }

        .actions {
          margin-top: 2rem;
        }

        button {
          background: #ececec;
          border: 0;
          border-radius: 0.125rem;
          padding: 1rem 2rem;
        }

        button + button {
          margin-left: 1rem;
        }
      `}</style>
    </Layout>
  );
};

export default Post;
```

>Postコンポーネントを更新し、APIルート経由での公開を処理するようにします。

このコードでは、先ほど実装したAPIルートにHTTP PUTリクエストを送信する役割を担う 'PublishPost' 関数をReactコンポーネントに追加します。また、コンポーネントの 'render' 関数を修正して、ユーザーが認証されているかどうかをチェックし、認証されている場合は、投稿の詳細画面にも**Publish**ボタンを表示するようにしました。

![The publish button shown for a post..](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/12.png)

>投稿用のPublishボタンが表示されました｡

ボタンをクリックしたら､投稿公開画面に遷移し､そこに投稿が表示されます！

>**注:** アプリを本番環境にデプロイすると、投稿公開画面が更新されるのはアプリ全体が再デプロイされたときだけです！これは、このビューのデータを取得するために 'getStaticProps' を使用して静的サイト生成 (SSG) を行っているためです。もし「すぐに」更新させたいのであれば、 'getStaticProps' を 'serverSideProps' に変更するか、Incremental Static Regeneration を使用することを検討してください。


# ステップ8. 削除機能の追加

最後は､ 'Post' レコードからユーザーが削除する機能の実装です｡まず､バックエンドでAPIルートハンドラを実装し、フロントエンドで新しいルートを利用するように修正します｡「公開」機能と同様のアプローチをとることになります。

'pages/api/post' ディレクトリに '[id].ts' ファイルを新規作成してください:

```
$ touch pages/api/post/[id].ts
```

>投稿を削除する新規APIルートを作成します｡

そして､作成したファイルに以下のコードを追加してください:

```ts
// pages/api/post/[id].ts

import prisma from '../../../lib/prisma';

// DELETE /api/post/:id
export default async function handle(req, res) {
  const postId = req.query.id;
  if (req.method === 'DELETE') {
    const post = await prisma.post.delete({
      where: { id: postId },
    });
    res.json(post);
  } else {
    throw new Error(
      `The HTTP ${req.method} method is not supported at this route.`,
    );
  }
}
```

>Prisma Clientを使用してデータベースを変更するためのAPIルートを追加します。

このコードは、 '/api/post/:id' URL経由で入ってくるHTTP DELETEリクエストを処理します。ルートハンドラーは、URLから 'Post' レコードの 'id' を取得し、Prisma Clientを使用してデータベース内のこのレコードを削除します。

この機能をフロントエンドで利用するためには、やはり投稿の詳細表示画面を修正する必要があります。 'pages/p/[id].tsx' を開き、 'publishPost' 関数のすぐ下に以下の関数を挿入してください:

```tsx
// pages/p/[id].tsx

async function deletePost(id: string): Promise<void> {
  await fetch(`/api/post/${id}`, {
    method: 'DELETE',
  });
  Router.push('/');
}
```

>Postコンポーネントを更新し、APIルート経由で削除を処理するようにします。

公開ボタンと同じように､認証ユーザーに**Delete**ボタンをレンダリングできます｡これを実現するには、**Publish**ボタンがレンダリングされる場所のすぐ下にある 'Post' コンポーネントの 'return' 部分を、次のコードに修正します:

```tsx
// pages/p/[id].tsx
        {
          !props.published && userHasValidSession && postBelongsToUser && (
            <button onClick={() => publishPost(props.id)}>Publish</button>
          )
        }
        {
          userHasValidSession && postBelongsToUser && (
            <button onClick={() => deletePost(props.id)}>Delete</button>
          )
        }
```

>PublisボタンとDeleteボタンのどちらかを表示するか決定するロジックを作成します｡

新しい下書きを作成し、その詳細表示に移動して、新しく表示された**Delete**ボタンをクリックすることで、この機能を試すことができるようになりました。

![The Delete button showing on the post page.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/13.png)

>Deleteボタンが投稿ページに表示されました｡

# ステップ9. Vercelにデプロイ

この最終ステップでは､アプリをGitHubリポジトリからVercelにデプロイします｡

デプロイ前に次のことが必要です:
- GitHubでもうひとつ新たにOAuth appを作成する
- 新規GitHubリポジトリを作成し､そこにこのプロジェクトをプッシュする

ステップ5-1を参考にし､もうひとつのOAuth appをGitHub UI上で作成してしてください｡

今回、**Authorization Callback URL**は、これからデプロイを行うVercelプロジェクトのドメインと一致させる必要があります。Vercelのプロジェクト名として、'blogr-nextjs-prisma' の前に、あなたの姓と名を付けます： 'FIRSTNAME-LASTNAME-blogr-nextjs-prisma' ｡
例えば､あなたの名前が "Jane Doe" なら､プロジェクト名は 'jane-doe-blogr-nextjs-prisma' にしてください｡

>**注:** デプロイ先のURLを一意にするため､あなたの名前を付けています｡

そのため、**Authorization Callback URL**は、 'https://FIRSTNAME-LASTNAME-blogr-nextjs-prisma.vercel.app/api/auth' に設定する必要があります。アプリケーションを作成したら、 '.env' ファイルを調整し、**Client ID**を 'GITHUB_ID' 環境変数に、**Client secret**を 'GITHUB_SECRET' 環境変数に設定します。 'NEXTAUTH_URL' 環境変数にはGitHubの**Authorization Callback URL**と同じ値: 'https://FIRSTNAME-LASTNAME-blogr-nextjs-prisma.vercel.app/api/auth' を設定する必要があります。

![Update the Authorization callback URL.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/14.png)

>Authorization callback URLを修正する｡

次に､ 'jane-doe-blogr-nextjs-prisma' のように名前を一致させたGitHubのリポジトリを作成します｡以下の3行のコマンドを実行し､リポジトリを作成してプッシュしてください｡

```
$ git remote add origin git@github.com:janedoe/jane-doe-blogr-nextjs-prisma.git
$ git branch -M main
$ git push -u origin main
```

>リポジトリをプッシュします｡

これで、新しいリポジトリが https://github.com/GITHUB_USERNAME/FIRSTNAME-LASTNAME-blogr-nextjs-prisma (例: https://github.com/janedoe/jane-doe-blogr-nextjs-prisma) に準備されたはずです。

GitHubのリポジトリができたので、それをVercelにインポートして、アプリをデプロイすることができます:


[![Deploy](https://vercel.com/button)](https://vercel.com/import/git?env=DATABASE_URL,GITHUB_ID,GITHUB_SECRET,NEXTAUTH_URL)

ここで、テキストフィールドにGitHubのリポジトリのURLを入力します:

![Import a git repository to Vercel.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/15.png)

>GitリポジトリをVercelにインポートする｡

**Continue**をクリックします｡この画面では､本番環境の環境変数を入力します｡

![Add environment variables to Vercel.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/16.png)

>Vercelに環境変数を入力します｡この画像に加えSECRET変数も設定してください｡

入力項目:
- 'DATABASE_URL': '.env' からこの値をコピーして入力してください
- 'GITHUB_ID': 本番用のGitHub OAuth appから**Client ID**をコピーして入力してください
- 'GITHUB_SECRET': 本番用のGitHub OAuth appから**Client secrets**をコピーして入力してください
- 'NEXTAUTH_URL': 本番用のGitHub OAuth appから**Authorization Callback URL**をコピーして入力してください
- SECRET: GITHUB_SECRETと同じものを入力してください｡Prismaが使用します｡

全ての環境変数が入力できたら､**Deploy**ボタンをクリックしてください｡あなたのアプリがVercelにデプロイされます｡準備ができたら､Vercelが成功画面を表示します:

![Your application deployed to Vercel.](https://vercel.com/docs-proxy/static/guides/nextjs-prisma-postgres/17.png)

>あなたのアプリがVercelにデプロイされました｡

**Visit**ボタンをクリックし､あなたの作成したフルスタックアプリを見ることができます🎉

# おわりに

このガイドでは、Next.js、Prisma、PostgreSQLを使用してフルスタックアプリケーションを構築し、デプロイする方法を学びました。もしこのガイドで問題が発生したり質問がある場合は、[GitHub](https://github.com/prisma/prisma/discussions)か、[Prisma Slack](https://slack.prisma.io/)に気軽に書き込んでください。
