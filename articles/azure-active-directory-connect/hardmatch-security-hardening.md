---
title: Microsoft Entra Connect のハードマッチの動作変更について
date: 2026-03-09 09:00:00
tags:
  - Microsoft Entra Connect
---

# Microsoft Entra Connect のハードマッチの動作変更について

こんにちは、 Azure Identity サポート チームの新倉です。

公開情報「Microsoft Entra のリリースとお知らせ」にて、セキュリティ強化のため 2026 年 5 月からMicrosoft Entra Connect (以下 MEC) のハードマッチの動作が変更されることが通知されました。

公開情報　
[Microsoft Entra のリリースとお知らせ - 一般提供 - ユーザー アカウントの引き継ぎを防ぐための Microsoft Entra Connect セキュリティ強化](https://learn.microsoft.com/ja-jp/entra/fundamentals/whats-new#general-availability---microsoft-entra-connect-security-hardening-to-prevent-user-account-takeover)

本ブログ記事では、動作の変更点と、変更後のハードマッチの手順をご紹介し、お問い合わせの多いご質問 (FAQ) について Q&A 形式で回答いたします。
今後、ハードマッチについてお問い合わせの多いご質問は適宜内容を追記していき、また公開情報では記載されていない点についても補足してまいります。


## 1. Microsoft Entra Connect のハードマッチの変更点について

### 1-1. 2026 年 5 月以降に変更される点

MEC のハードマッチは、オンプレミス AD から新しいオブジェクトを同期する際、 Microsoft Entra ID 上のアカウントと同期しようとしたオンプレミス AD のアカウントが一致するかを判断する属性として、 sourceAnchor を基に評価する方法を指します。

今回、2026 年 5 月以降に予定されている変更では、セキュリティ強化のため、既存オブジェクトと一致するかを判断する際に sourceAnchor の属性だけではなく OnPremisesObjectIdentifier 属性の検証が追加され、以下のすべての条件が成り立つ場合にのみハードマッチが成功するように動作が変わります。


- a. Microsoft Entra ID アカウントと同期しようとするオンプレミス AD の sourceAnchor が一致する
- b. テナントのプロパティ BlockCloudObjectTakeoverThroughHardMatch が Falseである
- **c.  OnPremisesObjectIdentifier 属性が空か、もしくは オンプレミスから入ってくる AD オブジェクトのオブジェクト識別子と一致している (New)**

**これまでは a. b. の条件を満たせばハードマッチが行われていましたが、今回の動作変更によって c. の条件が追加されます。**

一度でもオンプレミス AD からユーザー同期を行うと、 OnPremisesObjectIdentifier 属性には同期元ユーザーの object ID の値が設定されます。OnPremisesObjectIdentifier 属性に設定されている Object ID とは異なる Object ID を持つユーザーが同期されてきた場合、たとえ sourceAnchor が一致していても同期エラーが発生しハードマッチに失敗します。


### 1-2. ハードマッチの動作が変更された後のハードマッチ手順について

動作変更後もハードマッチを実施するには、 OnPremisesObjectIdentifier 属性が空である必要があるため、管理者が事前に Graph API を用いて同期済みユーザーの OnPremisesObjectIdentifier をクリアする必要があります。
ハードマッチを利用する際の シナリオをもとに、以下でご説明いたします。

補足: 本手順では sourceAnchor は mS-DS-ConsistencyGuid を指定した環境での手順をご案内しております。 mS-DS-ConsistencyGuid 以外の属性を sourceAnchor としている場合は、 sourceAnchor に指定した属性に置き換えてお読みください。

#### シナリオ: オンプレミス AD の非同期ユーザーと、既存の Entra ID ユーザーを紐づける場合

オンプレミス AD フォレストの移行や出向などの理由により、既存の Entra ID ユーザーに紐づく同期元のオンプレミスユーザーを変更する場合がございます。
このシナリオでは、Entra ID にある既存のユーザー A' について、同期元となっているオンプレミスユーザーをドメイン X にいるユーザー A からドメイン Y にいるユーザー B に交換します。

変更前:
ドメイン X On-prem user A -> Entra ID user A'
ドメイン Y On-prem user B

変更後:
ドメイン X On-prem user A
ドメイン Y On-prem user B -> Entra ID user A'

ハードマッチ手順：

1. Entra Connect の定期同期を無効にします。

実行する PowerShell :
```powershell
Set-ADSyncScheduler -SyncCycleEnabled $false
```

2. Entra ID ユーザー A' について、ハードマッチを実施できるよう Microsoft Graph API を使って onPremisesObjectIdentifier を null に設定します

> [!NOTE]
> onPremisesObjectIdentifier を null に設定する Graph API は Beta バージョンの API でのリリースが先行されます。v1.0 の API でのリリース時期は現時点で未定です。
> ユーザー A' のように同期済みのユーザーでも、SOA を変更することなく onPremisesObjectIdentifier を null にすることができます

必要なアクセス許可 :
User-OnPremisesSyncBehavior.ReadWrite.All と User.ReadWrite.All

実行する Graph API :
```http
PATCH https://graph.microsoft.com/beta/users/<UserId>
{ onPremisesObjectIdentifier: null }
```

Microsoft Graph PowerShell の場合のコマンド例 :
```powershell
Connect-MgGraph -Scopes 'User-OnPremisesSyncBehavior.ReadWrite.All','User.ReadWrite.All' -TenantId  contoso.onmicrosoft.com
Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/beta/users/12345678-2468-adcd-dcba-1234567890ab"
Invoke-MgGraphRequest -Method PATCH -Uri "https://graph.microsoft.com/beta/users/12345678-2468-abcd-dcba-1234567890ab" -Body '{"onPremisesObjectIdentifier": null }'
```

3. オンプレミスユーザー B を同期対象外の OU に配置し、mS-DS-ConsistencyGuid 属性を、オンプレミスユーザー A の mS-DS-ConsistencyGuid 属性と同じ値になるよう設定します

4. オンプレミスユーザー A を同期対象外 OU に移動します

5. 差分同期を 2 回実行し、 MEC 並びに Entra ID ユーザー A' を Entra ID 上から削除します。 Entra ID ユーザー A' は「削除済みユーザー」となります

実行する PowerShell :
```powershell
Start-ADSyncSyncCycle delta
```

6. オンプレミスユーザー B を同期対象の OU に移動します

7. 差分同期を実行し、Entra ID ユーザー A' にオンプレミスユーザー B の情報を同期します

実行する PowerShell :
```powershell
Start-ADSyncSyncCycle delta
```

8. ハードマッチが行われ、Entra ID ユーザー A' とオンプレミスユーザー B が紐づき、以後はオンプレミスユーザー B の変更内容が Entra ID ユーザー A' に同期されるようになります

9. Entra Connect の定期同期を有効に戻します。

実行する PowerShell :
```powershell
Set-ADSyncScheduler -SyncCycleEnabled $true
```

## 2.ハードマッチの動作の変更点についてのよくあるご質問とその回答 (FAQ)

今回のハードマッチの動作変更について、以下によくあるご質問をおまとめしております。

---
### <span style="color: blue; ">Q:</span> この変更は特定のバージョンの MEC でのみ有効になりますか？それとも古いバージョンにも適用されますか？

<span style="color: red; ">A:</span> 
今回の変更は Microsoft Entra ID のサービス側で行われる変更となり、OnPremisesObjectIdentifier 属性の検証はハードマッチ操作時にサービス側で実施されるため、MEC のバージョンには依存しません。
特に MEC のアップグレードは必要なく、2026 年 5 月以降、全ての MEC バージョンに適用されます。

---
### <span style="color: blue; ">Q:</span> OnPremisesObjectIdentifier 属性が一致するかを検証してハードマッチする新しい動作を停止することはできますか？

<span style="color: red; ">A:</span> 
いいえ、ハードマッチを検証する新しい動作を停止することはできません。

---
### <span style="color: blue; ">Q:</span> 以下の公開情報で、フラグを有効化するとありますが、フラグとは何ですか？
 
[一般提供 - ユーザー アカウントの引き継ぎを防ぐための Microsoft Entra Connect セキュリティ強化](https://learn.microsoft.com/ja-jp/entra/fundamentals/whats-new#general-availability---microsoft-entra-connect-security-hardening-to-prevent-user-account-takeover)
 
<span style="color: red; ">A:</span> 
BlockCloudObjectTakeoverThroughHardMatch を指します。
これはハードマッチを許可・無効化するフラグであり、以前から存在する設定であり、既定で False となっています。
ハードマッチ自体をブロックしたい場合は、この値を True に設定します。
このフラグを変更しても OnPremisesObjectIdentifier 属性を検証し、一致しない場合にハードマッチをブロックする新しい動作を止めることはできません。

---
### <span style="color: blue; ">Q:</span> この変更はいつから各テナントに適用されますか？
 
<span style="color: red; ">A:</span> 
2026 年 5 月から段階的に各テナントに適用されます。各テナントでの具体的な日程については公開されておりません。

---
### <span style="color: blue; ">Q:</span> BlockCloudObjectTakeoverThroughHardMatch の現在の設定を確認する方法を教えてください。
 
<span style="color: red; ">A:</span> 
以下のコマンドで確認が可能です。

```PowerShell
(Get-MgBetaDirectoryOnPremiseSynchronization).Features |select BlockCloudObjectTakeoverThroughHardMatchEnabled
```

参考：
[Get onPremisesDirectorySynchronization - Microsoft Graph v1.0 | Microsoft Learn](https://learn.microsoft.com/ja-jp/graph/api/onpremisesdirectorysynchronization-get?view=graph-rest-1.0&tabs=powershell)
 
[Get-EntraDirSyncFeature (Microsoft.Entra.DirectoryManagement) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.entra.directorymanagement/get-entradirsyncfeature?view=entra-powershell)


---
### <span style="color: blue; ">Q:</span> OnPremisesObjectIdentifier 属性が同期されていないユーザーには今回の変更は影響ありませんか？
 
<span style="color: red; ">A:</span>
はい、OnPremisesObjectIdentifier 属性が同期されていないユーザーに影響はありません。

---
### <span style="color: blue; ">Q:</span> 以前にハードマッチで同期されていたユーザーについて、今回の動作変更の影響はありますか？
 
<span style="color: red; ">A:</span>
この変更はハードマッチの実行時に評価されるため、既にハードマッチで同期済みのユーザーには影響はありません。

---
### <span style="color: blue; ">Q:</span> 2026 年 5月のロールアウト後も、BlockCloudObjectTakeoverThroughHardMatch を True に設定することは可能ですか？
 
<span style="color: red; ">A:</span>
はい、可能です。

---
### <span style="color: blue; ">Q:</span> サポートブログ [ハードマッチによる同期ユーザーの切り替え方法 (AD フォレスト移行 編) | Japan Azure Identity Support Blog](https://jpazureid.github.io/blog/azure-active-directory-connect/aadc_hardmatch/) の手順は変更されますか？
 
<span style="color: red; ">A:</span>


---
### <span style="color: blue; ">Q:</span> セキュリティ強化にともなう変更は Entra ID のサービス側の変更であって Entra Connect のバージョンに関係なく機能が展開される認識です。一方で、公開情報には最新バージョンへのアップグレードを推奨しているような記載があるのはなぜですか？
 
<span style="color: red; ">A:</span>
セキュリティ強化の恩恵を受けるには onPremisesObjectIdentifier が同期規則に追加されているバージョン (2.2.8.0 以降) の MEC にアップグレードする必要があります。
そのために最新のバージョンの MEC へのアップグレードを推奨しています。

---
### <span style="color: blue; ">Q:</span> 
ハードマッチによって同期元が切り替わったユーザーを確認する方法はありますか？
 
<span style="color: red; ">A:</span>
いいえ、確認することはできません。

ただしハードマッチによって同期元が変わったかを判断する一助となる方法として、 sourceAnchor を mS-DS-ConsistencyGUID としている場合に限定されますが、クラウドに同期されたユーザーの sourceAnchor と onPremisesObjectIdentifier を確認する方法が挙げられます。

mS-DS-ConsistencyGUID を sourceAnchor としている場合、 mS-DS-ConsistencyGUID が Null の状態で同期を行うとオンプレミスの Object Guid を基に sourceAnchor が生成され、生成された sourceAnchor が mS-DS-ConsistencyGUID にライトバックされます。

そのため、 mS-DS-ConsistencyGUID に意図的に別の値を入力した状態で同期しない限りは mS-DS-ConsistencyGUID と onPremisesObjectIdentifier は一致します。一致しない場合は、ハードマッチを用いた可能性があると判断可能です。
 
---
### <span style="color: blue; ">Q:</span> 
管理者ロールをもつクラウドユーザーに対するハードマッチの動作について、変更点はありますか？

<span style="color: red; ">A:</span>
2026年6月1日以降、Microsoft Entra IDは、Active Directoryの新しいユーザーオブジェクトと、特権ロールを持つ既存のクラウド管理のMicrosoft Entra IDユーザーオブジェクトをハードマッチングしようとすると、ブロックするようになります。
このブロックにより、クラウド管理のユーザーがすでにonPremisesImmutableId(sourceAnchor)を設定し、特権ロールが割り当てられている場合、Microsoft Entra Connect SyncまたはCloud Syncは、Active Directoryからの受信ユーザーオブジェクトとハードマッチングすることでそのユーザーの権限を乗っ取る試みができなくなります。

参考:
[Upcoming change – Microsoft Entra Connect security update to block hard match for privileged roles](https://learn.microsoft.com/en-us/entra/fundamentals/whats-new#upcoming-change--microsoft-entra-connect-security-update-to-block-hard-match-for-privileged-roles)
 
---


## 3.参考資料

[一般提供 - ユーザー アカウントの引き継ぎを防ぐための Microsoft Entra Connect セキュリティ強化](https://learn.microsoft.com/ja-jp/entra/fundamentals/whats-new#general-availability---microsoft-entra-connect-security-hardening-to-prevent-user-account-takeover)

[マッチングおよびハードマッチについて - Microsoft Entra Connect: 既存のテナントがある場合](https://learn.microsoft.com/ja-jp/entra/identity/hybrid/connect/how-to-connect-install-existing-tenant)

[ハードマッチによる Azure AD (Office 365) 上のユーザーをオンプレミス Active Directory ユーザーと紐付ける方法](https://jpazureid.github.io/blog/azure-active-directory-connect/upn-hard-match/)

[ハードマッチによる同期ユーザーの切り替え方法 (AD フォレスト移行 編) | Japan Azure Identity Support Blog](https://jpazureid.github.io/blog/azure-active-directory-connect/aadc_hardmatch/)

[Get onPremisesDirectorySynchronization - Microsoft Graph v1.0 | Microsoft Learn](https://learn.microsoft.com/ja-jp/graph/api/onpremisesdirectorysynchronization-get?view=graph-rest-1.0&tabs=powershell)
 
[Get-EntraDirSyncFeature (Microsoft.Entra.DirectoryManagement) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.entra.directorymanagement/get-entradirsyncfeature?view=entra-powershell)

