### 概要処理

1-パスワード再設定処理
- 分析ユーザーがパスワード期間切れの場合、ログインID（メールアドレス）あてに再設定通知を送信して、パスワード再設定する
  - パスワードのお忘れ場合のリンクをクリックして、パスワード再設定メール送信画面に遷移する
  - ログインID（メールアドレス）を入力する
  - URLを発行する
  - メールをクリックして、パスワード再設定画面に遷移する
  - パスワードを再設定して、パスワード再設定済みメールを送信する
  - データベースに使用済みトークンとパスワード変更ログを登録する

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': {
}}}%%

sequenceDiagram
    actor ユーザ
    participant Login as ログイン画面
    participant EmailForm as パスワード再設定メール送信画面
    participant m_stat_user as 分析ユーザマスタ
    participant m_company as 会社マスタ
    participant t_access_token_blacklist as アクセストークンブラックリスト
    participant Mailer as メールサーバ
    participant l_password_log as パスワードログ
    participant ChangePassword as パスワード再設定画面
    participant ActiveExpired as 有効期限切れ画面

    %% Step 1–3
    ユーザ->>Login: ログイン画面にアクセス
    Login-->>ユーザ: ログイン画面を表示
    ユーザ->>EmailForm: 「パスワードをお忘れの場合」リンクをクリック
    ユーザ->>EmailForm: ログインIDを入力し、「送信」をクリック
    activate EmailForm

    %% Step 4: バックエンド処理
    EmailForm->>m_stat_user: ログインIDより分析ユーザマスタを照会
    activate m_stat_user
    m_stat_user-->>EmailForm: 分析ユーザマスタの情報を返却
    deactivate m_stat_user

    alt 分析ユーザマスタが存在しない場合
    else 分析ユーザマスタが存在する場合
        EmailForm->>m_company: 分析ユーザマスタの会社IDより会社情報を照会
        activate m_company
        m_company-->>EmailForm: 会社情報を返却
        deactivate m_company
        EmailForm->>EmailForm: JWT形式の一時トークンを生成（有効時間10分、会社マスタに基づく）
        EmailForm->>t_access_token_blacklist: トークンと有効期限を保存
        activate t_access_token_blacklist
        t_access_token_blacklist-->EmailForm: 保存成功
        deactivate t_access_token_blacklist
        EmailForm->>Mailer: パスワード再設定リンクを含むメールを送信
        activate Mailer
        Mailer-->>EmailForm: メール送信成功
        deactivate Mailer
    end
    deactivate EmailForm

    EmailForm-->>ユーザ: 「パスワード再設定のご案内を送信いたしました。メールをご確認ください。」を表示
    ユーザ-->>ユーザ: メールを受信し、リンクをクリック
    ユーザ->>ChangePassword: メール内のURLをクリック
    activate ChangePassword

    ChangePassword->>t_access_token_blacklist: アクセストークンブラックリストをトークンで照会
    activate t_access_token_blacklist

    alt トークンが有効な場合
        ユーザ->>ChangePassword: 新しいパスワードを入力し、「送信」をクリック
        ChangePassword->>m_stat_user: パスワードを更新
        activate m_stat_user
        m_stat_user-->>ChangePassword: パスワード更新成功
        deactivate m_stat_user

        ChangePassword->>l_password_log: パスワード履歴を追加
        activate l_password_log
        l_password_log-->>ChangePassword: パスワード履歴追加成功
        deactivate l_password_log

        t_access_token_blacklist->>t_access_token_blacklist: 分析ユーザマスタの他トークンを無効化
        t_access_token_blacklist-->>ChangePassword: アクセストークン無効化成功

        ChangePassword->>Mailer: パスワード変更通知メールを送信
        activate Mailer
        Mailer-->>ChangePassword: メール送信成功
        deactivate Mailer

        ChangePassword-->>Login: ログイン画面にリダイレクト
        Login-->>Login: 「パスワードを再設定しました。」を表示
    else トークンが無効な場合
        t_access_token_blacklist-->>ActiveExpired: 「リンクが無効となっています。」を表示
        ActiveExpired-->>ユーザ: 「パスワード再設定のご案内を送信いたしました。メールをご確認ください。」リンクを表示
    end

    deactivate t_access_token_blacklist
    deactivate ChangePassword

```
