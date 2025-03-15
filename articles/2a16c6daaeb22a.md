---
title: "XユーザーのD&D風アラインメントチャートを試してみた"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Alignment", "AI", "X", "OpenAI", "Exa"]
published: false
---

# XユーザーのD&D風アラインメントチャートを試してみた

## はじめに

今、X(旧Twitter)上でバズっているD&D風の性格診断チャートが面白かったので解析してみました。

↓こちらで試せます。
[magic-x-alignment-chart.vercel.app](http://magic-x-alignment-chart.vercel.app)

![アラインメントチャートの画像](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/30621/4dfb7831-f0cb-4691-b014-67f7276c5d4a.png)

### D&D とは？

テーブルトークRPGの始祖 ダンジョンズ＆ドラゴンズ(D&D)です。そのD&Dのキャラクター作成時に性格(アラインメント)がありました。

*まだ持っている私物の写真*
![D&D本の写真](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/30621/5187faae-b205-46a0-909d-e400636bd84e.png)


![アラインメント表](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/30621/2f530238-f233-423c-a5a2-8dae408d03ee.png)

キャラクターの倫理観や行動傾向を「秩序-混沌」と「善-悪」の2軸で表現するシステムです。

このアラインメントチャートの考え方をX（旧Twitter）ユーザーに適用し、AIによる分析で自動的に位置付けを行うWebアプリです。


## 開発の経緯

このプロジェクトは、[@mdmathewdc](https://x.com/mdmathewdc)さんが投稿したアイデアに触発されたものです。

https://x.com/mdmathewdc/status/1899767815344722325

その後、[@rauchg](https://x.com/rauchg)さんがドラッグ＆ドロップ機能を実装し、

https://x.com/rauchg/status/1899895262023467035

[@f1shy-dev](https://x.com/vishyfishy2)さんがAI分析機能を追加して発展させたとのことです。

https://x.com/vishyfishy2/status/1899929030620598508

これらの実装をベースに、UIの改善やモバイル対応、パフォーマンス最適化が行われ、より多くの人が使いやすいアプリに仕上げられています。オープンソースコミュニティの力で、アイデアが実際のプロダクトに進化する過程が非常に興味深いプロジェクトだと言えるでしょう。


## アプリの特徴

1. **AIによる自動分析**
   - [Exa](https://exa.ai/) と OpenAI GPT4 を利用してユーザーのツイートを分析
   - 秩序(Lawful) - 混沌(Chaotic)、善(Good) - 悪(Evil) の2軸でユーザーの傾向を評価
   - 分析結果に基づいて自動的にグリッド上に配置

2. **インタラクティブな操作性**
   - ドラッグ＆ドロップによる直感的な位置調整
   - AI分析結果はロック機能付き
   - ランダム配置機能で手動調整も可能

3. **技術スタック**
   - Next.js（App Router）
   - TypeScript
   - Framer Motion（アニメーション）
   - Vercel AI SDK
   - IndexedDB（ローカルストレージ）
   - Upstash Redis（キャッシュ）

## 技術的な実装ポイント

### 1. レスポンシブ設計

```typescript
useEffect(() => {
  const checkMobile = () => {
    setIsMobile(window.innerWidth < 768);
  };
  
  checkMobile();
  window.addEventListener("resize", checkMobile);
  return () => window.removeEventListener("resize", checkMobile);
}, []);
```

アプリは、モバイルからデスクトップまで、様々なデバイスで最適な操作感を実現するために、画面サイズに応じた動的なレイアウト調整が実装されています。特にタッチ操作とマウス操作の両方に対応させるのは難しい課題だったようですが、Framer Motionの機能を活用してスムーズな操作感が実現されています。

### 2. AIによるアラインメント分析

```typescript
export async function analyzeAlignment(username: string) {
  try {
    const tweets = await fetchUserTweets(username);
    
    // Exaを使用して分析
    const analysis = await exaClient.run(
      `ユーザー ${username} のツイート内容から、D&Dのアラインメント（秩序-混沌、善-悪）を判断してください。
       以下のツイートを参考にしてください:
       ${tweets.join("\n")}`
    );
    
    return {
      lawChaos: analysis.lawChaosScore, // -5(混沌)～+5(秩序)
      goodEvil: analysis.goodEvilScore,  // -5(悪)～+5(善)
      confidence: analysis.confidence,
      explanation: analysis.reasoning
    };
  } catch (error) {
    console.error("アラインメント分析エラー:", error);
    throw new Error("ユーザーの分析中にエラーが発生しました");
  }
}
```

このアプリでは、Exaのコンテキスト理解能力を活用して、ユーザーのツイート履歴からパーソナリティや行動傾向を分析し、アラインメントチャート上の最適な位置を算出しています。単語の出現頻度だけでなく、文脈や感情も含めた総合的な分析が可能になっているようです。

#### プロンプト

```ts
// GPT-4に渡すメッセージを構築
const messages = [
  {
    role: "system",
    content: dedent`
    与えられたTwitterユーザーのツイートを分析し、D&Dスタイルのアラインメントチャート上の位置を判定してください。
    
    秩序-混沌の軸：
    - 秩序 (-100): 規則、伝統、社会規範に従う。伝統、忠誠、秩序を重視。
    - 中立 (0): 規則と自由に対してバランスの取れたアプローチ
    - 混沌 (100): 因習に反発し、個人の自由を重視。規則や伝統に関係なく自分の信念に従う
    
    善-悪の軸：
    - 善 (-100): 利他的、思いやりがあり、他者を優先
    - 中立 (0): 自己利益と他者への配慮がバランスを保つ
    - 悪 (100): 利己的、操作的、他者に危害を加える。財款、憤怒、権力欲に動機付けられる
    
    これらのツイートのみに基づいて、ユーザーのアラインメントを数値で評価してください。どの極端にも振れることを恐れないでください！

    これは楽しむためのものなので、ユーザーの特徴がわずかでも見られれば、それを誤張して構いません。例えば、悪の要素があれば、それを強調してください！全員を「混沌の中立」にしたくはありません。ただし、必ずしも混沌の特徴を誤張する必要はなく、より願著な秩序や善/悪の特徴を誤張することもできます。楽しく分析してください。
    
    説明は冗長になりすぎないようにしつつ、判断の理由を示してください。ユーザーの特徴や発言、プロジェクトなどを具体的に言及すると、よりパーソナライズされた分析になります。
  `.trim()
  },
  {
    role: "user",
    content:
      dedent`Username: @${username}

<user_profile>
${profile_str}
</user_profile>

<user_tweets filter="top_100">
${tweetTexts}
</user_tweets>`.trim()
  }
] satisfies CoreMessage[]
```

### 3. データ永続化とキャッシュ戦略

```typescript
// IndexedDBを使ったデータ永続化
const saveToLocalDB = async (placement: StoredPlacement) => {
  if (!db) return;
  try {
    const tx = db.transaction("placements", "readwrite");
    const store = tx.objectStore("placements");
    await store.put({
      ...placement,
      timestamp: new Date()
    });
    await tx.done;
  } catch (error) {
    console.error("保存エラー:", error);
  }
};

// Upstash Redisを使ったAPIレスポンスのキャッシュ
export async function cachedFetchTweets(username: string) {
  const cacheKey = `tweets:${username}`;
  
  // キャッシュをチェック
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached as string);
  
  // APIから取得
  const tweets = await fetchFromTwitterAPI(username);
  
  // キャッシュに保存（24時間有効）
  await redis.set(cacheKey, JSON.stringify(tweets), { ex: 86400 });
  
  return tweets;
}
```

ユーザー体験を向上させるため、IndexedDBを使ってブラウザ内に配置情報を永続化し、Upstash Redisを活用してAPI呼び出しの結果をキャッシュすることで、無駄なAPIリクエストを減らしてパフォーマンスを最適化する工夫が施されています。これにより、Twitter APIの制限にも対応しているようです。


## 実装で苦労していると思われる点と解決策

### 1. Twitter APIの制限対応

Twitter（現X）APIの制限は厳しく、頻繁にリクエストを送ると簡単にレート制限に引っかかるという問題があります。この問題を解決するために以下の対策が講じられているようです：

- Upstash Redisを使ったアグレッシブなキャッシュ戦略
- バックオフとリトライのロジック
- エラー発生時のユーザーフレンドリーな代替フロー

これらの対策により、APIの制限内でも快適にアプリを使用できるよう工夫されています。

### 2. モバイルUXの最適化

```typescript
const TouchControls = ({ onReset, onRandom }) => {
  return isMobile ? (
    <div className="fixed bottom-4 left-0 right-0 flex justify-center gap-3 z-10">
      <Button variant="outline" onClick={onReset}>
        <RefreshCw size={16} className="mr-2" />リセット
      </Button>
      <Button variant="outline" onClick={onRandom}>
        <Shuffle size={16} className="mr-2" />ランダム配置
      </Button>
    </div>
  ) : null;
};
```

タッチデバイスでは、ドラッグ操作とスクロール操作が競合する問題があったようです。これを解決するために：

- タッチイベントの伝播制御
- モバイル向けの専用コントロールUIの追加
- 要素サイズの最適化

といった対策が施され、モバイルとデスクトップの両方で快適に使えるUIが実現されています。

## まとめ

このプロジェクトでは、モダンなWeb技術とAIを組み合わせることで、Xユーザーのパーソナリティを視覚的に表現する新しい方法が提案されています。技術的な挑戦はいくつかあったようですが、それらを乗り越えることで、面白くて実用的なWebアプリが作り上げられています。

## リンク
- デモサイト：[magic-x-alignment-chart.vercel.app](https://dub.sh/magic-x-alignment-chart/)
- ソースコード：[f1shy-dev/x-alignment-chart](https://github.com/f1shy-dev/x-alignment-chart)