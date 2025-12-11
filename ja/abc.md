
──────────────────
既存のページで修正した方が良い点
──────────────────

1 フォームの基本属性を明示する
・form タグに method と action を付ける
・method="post"
・action="（デプロイした GAS の Web アプリ URL）"
→ これで「このフォームの送信先は GAS」「POST で送る」が明確になる。

2 返信用メールアドレス項目を追加する
・ご担当者メールアドレスを必須項目として追加
・input type="email" id="email" name="email" required
・必要に応じてご担当者名（person）も追加
→ 管理側がそのまま「返信」できる形にするために必須。

3 name 属性の統一・確認
・全ての項目に name が付いているか確認（GAS は name ベースで受け取る）
・company, country, person, email, website, category, market, sales, goals, contact-method
→ このフォームはほぼ揃っているので、追加項目と contact-method のスペルだけ意識しておけば十分。

4 HTML 側の最低限バリデーション
・required を付ける項目の整理
・company, country, email は required
・その他は任意（今のままでも OK）
→ 本番の信頼性は GAS 側のバリデーションで担保しつつ、フロント側はユーザーの入力ミス防止用。

5 完了メッセージの導線
・送信後に「ありがとうございました」系のメッセージを出せるようにする
・一旦は「送信後に専用の thanks ページへリダイレクト」でも良いし
・後で fetch ベースに変えるなら、送信完了メッセージ用のエリアを作っておいても良い
→ UX 的に「送れたのかどうか」が分かるようにしておく、という意味。

──────────────────
GAS によって追加する機能
──────────────────

1 Web エンドポイントとしての受付機能
・doPost(e) でフォームからの POST を受け取る
・e.parameter.company などで各項目を取得
→ 静的ページからのフォーム送信を「メール送信処理」に橋渡しする役。

2 サーバー側バリデーション
・必須項目のチェック
・company, country, email が空でないか
・メールアドレス形式チェック
・簡易な正規表現で OK
・文字数制限
・goals が異常に長過ぎる場合は弾く
→ 攻撃やスパムっぽい入力に対する「最後の防波堤」。

3 管理者向け通知メール送信
・宛先: あなたの管理用メールアドレス
・件名: 「[日本市場テスト相談] 会社名 / 国」 形式
・本文: 送信された全項目を整形して一覧表示
・Reply-To: ユーザーの email
→ 届いたメールに普通に「返信」するだけでコンタクトが取れる。

4 ユーザー向け自動サンクスメール
・宛先: フォームで入力された email
・件名: 「日本市場テストのご相談ありがとうございます（JBoundex）」 など
・本文:
・受付完了の挨拶
・送信内容のサマリ（会社名・国・goals など）
・返信までの目安時間
・contact-method に応じた一文（メール / ビデオ通話 / 電話）
→ 「ちゃんと届いたか不安」を消して、信頼感を出す部分。

5 スパム・攻撃対策のロジック
・同じ IP（または同じ email）から短時間に連打された場合の簡易制限
・スプレッドシートにログを残して、一定条件を超えたら無視するなど
・怪しい文字列やタグの無害化
・本文に含まれる HTML っぽい文字列をそのままテキスト扱いにする
→ 完全防御ではないけど、「ただの素通しフォーム」より一段階守りを厚くする役割。

6 ログ保存（任意だがかなり有用）
・送信された内容をスプレッドシートに 1 行ずつ保存
・タイムスタンプ
・各フォーム項目
・送信元 IP など（取れる範囲で）
→ 後から「どの国からどんな相談が多いか」「CV 計測」などの分析にも使える。

7 レスポンス処理
・正常系
・HTML を返して「サンクスページ」へ誘導
・あるいは JSON {success: true} を返して、フロント JS で完了表示
・異常系
・エラー内容をシンプルなメッセージにして返却（内部情報は出さない）
→ UX とセキュリティ両面で「見せる情報」と「隠す情報」を分ける役。

──────────────────

この二つをやれば、

・フロント側は「今の静的ページをちょっとだけ修正」
・裏では GAS が「サーバーとして振る舞ってくれる」

という構成になるので、まさに

「LPレベルの静的サイトだけど、ちゃんとした問い合わせフローとセキュリティを持っているフォーム」

にできます。


なお、以下はGASで作成予定のコードです。

// 管理者側で受け取るメールアドレス
const ADMIN_EMAIL = "your-admin@example.com";

// 件名の接頭語
const SUBJECT_PREFIX = "[日本市場テスト相談]";

// 目安の返信時間
const REPLY_DEADLINE_TEXT = "通常二営業日以内にご連絡いたします。";

// メインの入口
function doPost(e) {
  try {
    if (!e || !e.parameter) {
      return createErrorHtml("不正なリクエストです。");
    }

    const p = e.parameter;

    // パラメータ整形
    const company = (p.company || "").trim();
    const country = (p.country || "").trim();
    const person = (p.person || "").trim();
    const email = (p.email || "").trim();
    const website = (p.website || "").trim();
    const category = (p.category || "").trim();
    const market = (p.market || "").trim();
    const sales = (p.sales || "").trim();
    const goals = (p.goals || "").trim();
    const contactMethod = (p["contact-method"] || "").trim();

    // バリデーション
    const errors = [];

    if (!company) {
      errors.push("会社名は必須です。");
    }
    if (!country) {
      errors.push("国は必須です。");
    }
    if (!email) {
      errors.push("ご担当者メールアドレスは必須です。");
    } else if (!isValidEmail(email)) {
      errors.push("メールアドレスの形式が正しくありません。");
    }

    // 目標文が極端に長い場合はスパム扱い
    if (goals && goals.length > 4000) {
      errors.push("メッセージが長すぎます。分割して送信してください。");
    }

    if (errors.length > 0) {
      const msg = "入力内容に問題があります。" +
        "<br>" +
        errors.map(function (m) { return "・" + m; }).join("<br>");
      return createErrorHtml(msg);
    }

    // 管理者向けメール送信
    sendAdminMail({
      company: company,
      country: country,
      person: person,
      email: email,
      website: website,
      category: category,
      market: market,
      sales: sales,
      goals: goals,
      contactMethod: contactMethod
    });

    // ユーザー向け自動返信
    sendUserThanksMail({
      company: company,
      country: country,
      person: person,
      email: email,
      website: website,
      goals: goals,
      contactMethod: contactMethod
    });

    // 正常終了レスポンス
    return createSuccessHtml();

  } catch (err) {
    Logger.log(err);
    return createErrorHtml("送信処理中にエラーが発生しました。時間をおいて再度お試しください。");
  }
}

// メールアドレス形式チェック
function isValidEmail(email) {
  var re = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return re.test(email);
}

// 管理者向け通知メール
function sendAdminMail(data) {
  var subject = SUBJECT_PREFIX + " " + data.company + " / " + data.country;

  var lines = [];

  lines.push("以下の内容で日本市場テストの相談がありました。");
  lines.push("");
  lines.push("会社名: " + data.company);
  lines.push("国: " + data.country);
  lines.push("ご担当者名: " + (data.person || "(未入力)"));
  lines.push("ご担当者メールアドレス: " + data.email);
  lines.push("Webサイト / Amazon URL: " + (data.website || "(未入力)"));
  lines.push("");
  lines.push("商品カテゴリ: " + (data.category || "(未選択)"));
  lines.push("現在の主要市場: " + (data.market || "(未入力)"));
  lines.push("月間売上(概算): " + (data.sales || "(未選択)"));
  lines.push("");
  lines.push("日本市場での目標:");
  lines.push(data.goals || "(未入力)");
  lines.push("");
  lines.push("ご希望の連絡方法: " + formatContactMethod(data.contactMethod));
  lines.push("");
  lines.push("送信日時: " + Utilities.formatDate(new Date(), "Asia/Tokyo", "yyyy-MM-dd HH:mm:ss"));

  var bodyText = lines.join("\n");

  var htmlBody = bodyText
    .replace(/\n/g, "<br>");

  GmailApp.sendEmail(ADMIN_EMAIL, subject, bodyText, {
    name: "日本市場テスト相談フォーム",
    replyTo: data.email,
    htmlBody: htmlBody
  });
}

// ユーザー向け自動返信
function sendUserThanksMail(data) {
  var to = data.email;
  var subject = "日本市場テストのご相談ありがとうございます（JBoundex）";

  var personName = data.person || "ご担当者様";

  var lines = [];

  lines.push(personName + " 様");
  lines.push("");
  lines.push("この度は日本市場テストに関するご相談をお送りいただき、ありがとうございます。");
  lines.push("以下の内容でお問い合わせを受け付けました。");
  lines.push("");
  lines.push("会社名: " + data.company);
  lines.push("国: " + data.country);
  lines.push("Webサイト / Amazon URL: " + (data.website || "(未入力)"));
  lines.push("");
  lines.push("日本市場での目標:");
  lines.push(data.goals || "(未入力)");
  lines.push("");
  lines.push("ご希望の連絡方法: " + formatContactMethod(data.contactMethod));
  lines.push("");
  lines.push(REPLY_DEADLINE_TEXT);
  lines.push("お急ぎの場合は、このメールに直接ご返信ください。");
  lines.push("");
  lines.push("JBoundex");
  lines.push("日本市場テスト支援");

  var bodyText = lines.join("\n");

  var htmlBody = bodyText
    .replace(/\n/g, "<br>");

  GmailApp.sendEmail(to, subject, bodyText, {
    name: "JBoundex",
    htmlBody: htmlBody
  });
}

// 連絡方法の表示用整形
function formatContactMethod(method) {
  if (method === "email") {
    return "メール";
  }
  if (method === "video") {
    return "ビデオ通話";
  }
  if (method === "phone") {
    return "電話";
  }
  return "未選択";
}

// 正常終了時のレスポンス
function createSuccessHtml() {
  var html = ""
    + "<html><head><meta charset='UTF-8'><title>送信完了</title></head>"
    + "<body style='font-family:sans-serif; padding:24px; text-align:center;'>"
    + "<h1>送信が完了しました</h1>"
    + "<p>日本市場テストに関するご相談を受け付けました。</p>"
    + "<p>" + REPLY_DEADLINE_TEXT + "</p>"
    + "<p><a href='/'>トップページに戻る</a></p>"
    + "</body></html>";

  return HtmlService.createHtmlOutput(html);
}

// エラー時のレスポンス
function createErrorHtml(message) {
  var html = ""
    + "<html><head><meta charset='UTF-8'><title>エラー</title></head>"
    + "<body style='font-family:sans-serif; padding:24px; text-align:center;'>"
    + "<h1>送信に失敗しました</h1>"
    + "<p>" + message + "</p>"
    + "<p>時間をおいて再度お試しください。</p>"
    + "<p><a href='javascript:history.back();'>入力ページに戻る</a></p>"
    + "</body></html>";

  return HtmlService.createHtmlOutput(html);
}


