# Review resolution contract

Repository: Saber5656/famchat; PR #58

このファイルは既存のBot review findingに対する文書レベルの対応契約である。各節のresolutionは後続実装が満たすべき規範であり、focused verificationはresolve前に実装時点で実施する検証条件を示す。ここで実装・テスト・CI・実機検証を実行済みとは主張しない。Bot reviewの再triggerは行わず、repository full validationは後続の実装gateで実施する。

## Thread PRRT_kwDOTNkIOs6QDp1A

### Scope direct-room dedupe by space

**Normative resolution**

direct_keyのunique scopeへspace_idを含め、同一space内のuser pairだけをdedupeする。find-or-createとunique constraintを同じtenant scopeで原子的に実施し、別spaceのdirect roomを返さない。

**Focused verification before resolving this thread:**

同じadult pairを2つのspaceでcreateDirectし、2 roomが作られ、同一space内の並列createだけが同じroomへ収束することを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1B

### Count unread messages when no read pointer exists

**Normative resolution**

last_read_message_id=NULLは何も読んでいない状態として扱い、unread queryはNULL OR id>pointerで新規memberの既存messageをcountする。最初のreadでpointerを正しく設定する。

**Focused verification before resolving this thread:**

pointer NULL、最古ID、途中ID、最新IDをfixture化し、unread countとread後のcountが期待どおりになることをSQL integration testで確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1C

### Enforce room access when serving message media

**Normative resolution**

message attachmentのmedia routeはspace membershipだけでなくclaimed messageのroom accessを認可する。adult-only/direct roomはroom memberだけ、board mediaはboard/space policyに従う。attachment IDを知っているだけでは取得不可とする。

**Focused verification before resolving this thread:**

同一spaceの非member、room member、guardian/child、board attachmentについてGETを実行し、各room policyどおりのstatus/bodyになることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1E

### Target revocation events to the revoked session

**Normative resolution**

single-session revokeのsession.revoked eventはrevoked sessionだけへ送信し、同userの他のvalid sessionをlogoutさせない。user-wide revokeは別イベントとして明示的に全sessionへ送る。

**Focused verification before resolving this thread:**

同一userの2 sessionでsingle revokeとglobal revokeを実行し、socket room/event payload/disconnect対象がそれぞれsession単位・user単位に一致することを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1G

### Remove demoted guardians from guardian channels

**Normative resolution**

guardianからadultへdemoteしたとき、既存socketをspace:<id>:guardiansからleaveさせ、guardian-only eventを受信できないようにする。promotion/reconnect時のjoinとspace removalも同じre-evaluationへ含める。

**Focused verification before resolving this thread:**

接続中guardianをdemoteし、guardian channelのbroadcastが届かず一般space eventだけ届くこと、再promotion/reconnectで正しくjoinすることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1I

### Delete originals before marking attachments ready

**Normative resolution**

quarantine originalの削除をattachment status=readyより先に行うか、ready transition時にleftover cleanup obligationを原子的に記録してretryする。ready rowがq/ objectを無期限保持しない。

**Focused verification before resolving this thread:**

delete成功、delete一時失敗、worker killを各段階で注入し、readyへの遷移順序とretry cleanupが成立すること、ready+leftoverが最終的に残らないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1J

### Suppress quiet-hour content feed rows

**Normative resolution**

quiet childのcontent notificationはpushだけでなくin-app feed row、WS event、payload excerptも抑制する。quiet-exemptなsystem/safety notificationだけを明示allowlistで通す。

**Focused verification before resolving this thread:**

quiet on/off、message/board content、system safety notificationをfixture化し、feed/WS/pushの全channelでcontent漏えいがなく、allowlistだけが届くことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1K

### Exclude reported guardians from resolution mutations

**Normative resolution**

reported user/content authorがguardianの場合、queue/WS/notifyだけでなくresolve/dismiss mutationも対象guardianから拒否する。report id漏洩時もNOT_FOUNDまたは権限拒否としてstatusを変更しない。

**Focused verification before resolving this thread:**

guardian本人、別guardian、moderatorが同じreportをresolve/dismissし、reported guardianだけが拒否され、正当なmoderator操作とaudit eventが維持されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1L

### Start Mailpit in the integration workflow

**Normative resolution**

integration workflowへMailpit service/containerとhealthcheck、SMTP_URL、必要ならAPI URLを追加し、auth reset suiteが実際のMailpitへ接続する。service unavailableはjobを成功扱いにしない。

**Focused verification before resolving this thread:**

clean CI-like networkでPostgres/Redis/MinIO/Mailpitを起動し、password reset emailがMailpit API/inboxで取得できること、healthcheck前のtest開始が抑止されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1N

### Avoid retaining deleted objects in backups

**Normative resolution**

backupはtimestamped immutable snapshot+manifestを基本とし、mirrorを使う場合はsourceにないobjectをremoveする明示flagと整合するDB manifestを同一backup単位で保存する。削除済みchild/mediaがrestoreで復活しない。

**Focused verification before resolving this thread:**

live bucketからobjectを削除してbackupを更新し、manifestとrestore結果にobjectが残らないこと、同時刻のDB dumpとの整合性と誤削除防止を確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1P

### Enforce attachment ownership with one atomic claim

**Normative resolution**

message imageとboard imageのXOR ownershipを同じattachment row/claim fieldまたは同一transaction lockで原子的に確定する。message_idとboard_post_idを同時に非NULLにできず、競合側はretry/CONFLICTになる。

**Focused verification before resolving this thread:**

同一attachmentをsendImage/createPostから並列claimし、一方だけが成功してもう一方がconflictとなること、DB rowが必ず一つのownerだけを持つことをtransaction testで確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1S

### Key unauthenticated account limits by credential

**Normative resolution**

unauthenticated login/reset rate limitのaccount keyはsession user idではなくnormalized email/login credentialへ結びつける。no-oracle responseを維持し、authenticated routeではsession scopeを使う。

**Focused verification before resolving this thread:**

同一emailへの複数IP、異なるemail、未認証requestでlimitが期待どおり共有/分離されることを確認し、存在有無がresponseから推測できないことを検証する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1T

### Make adult re-joins reactivate removed memberships

**Normative resolution**

invite acceptはunique membership rowがstatus=removedのadultを検出し、insertではなくatomicにactiveへreactivate/updateする。child等のroleと不正space accessは別途拒否する。

**Focused verification before resolving this thread:**

active/removed/absent membershipと並列acceptをfixture化し、removed adultが一行のまま再参加でき、duplicate rowや二重roleがないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1U

### Fold leet symbols before stripping separators

**Normative resolution**

normalization pipelineはleet symbols @/$等を意図した文字へfoldしてからseparator strippingを行うか、separator ruleからleet対象を除外する。b@d/a$sの双方が指定term policyに従って同じ比較結果になる。

**Focused verification before resolving this thread:**

plain、leet、separator混在、case/Unicode variantをfixture化し、fold順序による文字欠落がなく、false negative/positiveの境界がregistryと一致することを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1W

### Make notification fanout retries idempotent

**Normative resolution**

fanoutはrecipient+event/type等のstable idempotency keyをDB unique constraint/upsertで保証する。部分成功後のBullMQ retryは既存feed row/WS deliveryを重複作成せず、transport retryとpersistence retryを分離する。

**Focused verification before resolving this thread:**

recipientの一部insert後にworker failureを注入してretryし、feed rowが一つだけであること、WS duplicateが契約どおり抑止され、別eventは独立して届くことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1X

### Let no-VAPID instances build the web app

**Normative resolution**

VAPID public keyをoptional/empty-safeにし、push disabled instanceでもweb buildとUI表示が成功する。subscribe/registerはkey absent時にno-opまたは明示disabled stateを返し、secretを要求しない。

**Focused verification before resolving this thread:**

VAPID env有り/無しでclean buildを行い、無しでもwebPublicEnvSchemaとsubscribe codeが失敗せずdisabled UIになること、有りでは通常のsubscriptionが動くことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkIOs6QDp1Z

### Include room routes in iOS app-link association

**Normative resolution**

AASA pathsへ/s/*と必要なroom/board/notification deep-link routesを追加し、web/pushで使うroute contractと一致させる。invite/linkの既存associationは維持する。

**Focused verification before resolving this thread:**

代表的なroom、board、notification、invite URLをAASA validatorとiOS universal-link fixtureで検査し、room routeがSafariへ落ちずapp associationへ解決されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。
