---
layout: post
title: 'macOS App Notarization'
date: 2021-03-14
author: Pofat
color: rgb(233,202,200)
cover: '/assets/app_notarization.jpg'
tags: Swift macOS Notarize
---

自從 2019 年六月一號後，所有新發佈到 macOS 10.15 以上版本的 App 都註定需要先~~交保護費給水果~~註冊 Apple Developer Program，因為一定要先經過一個叫做 App Notarization 的手續，而其中會需要由合法 Apple Developer Prgoram 所註冊的 Developer ID 憑證來簽你所發佈的 App。

先前情提要一下，在 macOS 上發佈 App 有兩種方式，一個是透過 App Store，基本上跟 iOS 大同小異；另一個則是直接把檔案丟給別人讓別人去裝，不透過 App Store 下載，但這種方式預設需要用一種叫做 Developer ID 的憑證去簽署散佈的 App/安裝檔，App Notarization 就是針對後者，你在發佈要先通過這個流程，別人拿到你的檔案時才不會打不開。因為 macOS 10.15 後上面有個 Gatekeeper 會阻擋未通過 Notarization 的 App 安裝/執行，況且系統偏好設定裡的「安全性與隱私權」中，允許執行的 App 來源選預設只剩下 App Store 與「已識別的開發者」，這裡指的就是通過 Notarization 的 App。

註：你仍能夠強制在安全性設定允許系統執行任何來源的 App ，只要在 Terminal 裡執行 `sudo spctl --master-disable`，再進一次安全性設定就會看到多出「從任何來源」的選項，反之要復原預設則是 `sudo spctl --master-enable`。

## 準備工作

基本上 Xcode 裡附的 Archive Organizer 已經可以幫你處理絕大多數的工作，詳可參考 Apple [文件](https://help.apple.com/xcode/mac/current/#/dev88332a81e)，有滿大的可能你並不是用 Xcode 來開發或 build 出最終的成品，而是仰賴 command line 來達成，所以介紹一下整個流程以及完整執行這項工作的 script。基本上主要工作就三項：上傳、確認驗證結果以及裝訂（Staple），每個步驟我們都可以用 script 來完成，不過在作業前需要準備以下幾件事：

* 準備 Developer ID 的憑證，App 和裡面各個 Plug-In 與 binary 都要用 `Developer ID Application`簽，如果有 pkg 則使用 `Devloper ID Installer` 來簽
* 請先確認 Xcode Command Line Tool 已經安裝完成（由 `xcode-select -p`）
* 先到 Apple ID 帳號管理頁面生成 [「App 專用密碼」](https://support.apple.com/zh-tw/HT204397)
* 為避免把密碼明文寫在你的 script 裡，我們可以把密碼存到 Keychain 先：`xcrun altool --store-password-in-keychain-item "AC_PASSWORD" -u "$your_applie_id" -p "$your_one_time_password"`，請注意這裡的 `AC_PASSWORD` 是用於存在 Keycahin 裡的 item 名稱，你也可以指名其它，以下提到都統一使用 `AC_PASSWORD`
* 取得 Provider Short Name： 執行 `xcrun altool --list-providers -u "$your_apple_id" -p "@keychain:AC_PASSWORD"` 後在 `ProviderShortName` 欄位裡的值，通常和你的 Team ID 相同，請注意要選對 Team
* 請確認 App 裡所有的 Plug-In，執行檔與其它所有 Binary 都務必用 `Developer ID Application` 憑證正確簽署

## Xcode 的設定

首先自己下 script build 的話，要先到 Xcode 裡 Project 的 Build Settings 中，在 `Other Code Signing Flags` 中加入 `--timestamp`，因為你必需把簽署的時間帶入才能通過 Notarization。另外值得一提的是，如果你的專案是在 Xcode 10 之前建立的，你需要再多加一個 `--options=runtime` 以啟用在 macOS 10.14 加入的 Hardened Runtime，Xcode 10 後預設是會幫你開的就可以不用，沒開的話不會過。

## 上傳你的 App

首先，要先搞清楚要上傳什麼檔，一般 macOS App 發佈可能是個 MyApp.zip，MyApp.pkg 或 App 或 pkg 放在 DMG 裡，無論如何上傳最外層的那個檔案就對了。接下來上傳只要執行：

{% highlight shell %}
xcrun altool --notarize-app \
		--primary-bundle-id "$some_bundlie_id" \
		--username "$your_applie_id" \
		--password "@keychain:AC_PASSWORD" \
        --asc-provider "$provider_short_name" \
        --file "MyApp.dmg" &> notarization_log
{% endhighlight %}

注意這裡的 `prmiary-bundle-id` 其實可以隨便填，只是給個人識別用，成功後你會在輸出 log 的檔案裡取得一個 UUID，請握好它。

## 確認結果

經過我的測試幾乎上傳後兩三分鐘內都會完成，執行 `xcrun altool --notarization-info "$uuid" -u "$your_applie_id" -p "@keychain:AC_PASSWORD"`，可以得到結果，可能是還在進行中，成功(success)或失敗 (invalid)，如果失敗可以在回應的 `LogFileURL` 裡看到失敗的原因，可以寫個 script ，不時回傳檢查結果。

## 裝訂 App

完成後可以用 `xcrun stapler staple "YourApp.dmg"` 來裝訂成果，請記得一樣是裝訂最外層容器那個檔案即可，接下來就能安心散佈了，只是每次發佈新版前都要再走一次這流程。

## 一次完成

建立一個 notarize 的 shell script file 如下，用法 `./notarize $your_apple_id $prvoider_short_name $app_file_path`

{% highlight shell %}
#!/bin/sh
echo "Upload $3 for notarization"

xcrun altool --notarize-app \
		--primary-bundle-id "$some_bundlie_id" \
		--username "$1" \
		--password "@keychain:AC_PASSWORD" \
        --asc-provider "$2" \
        --file "$3" &> an_log

uuid=`cat an_log | grep -Eo '\w{8}-(\w{4}-){3}\w{12}$'`
while true; do
    echo "Checking notarization for $3..."
 
    xcrun altool --notarization-info "$uuid" -u "$2" -p "@keychain:AC_PASSWORD" &> result_tmp 
    result=`cat result_tmp`
    s=`echo "$result" | grep "success"`
    i=`echo "$result" | grep "invalid"`
    if [[ "$s" != "" ]]; then
        echo "Notarization done!"
        xcrun stapler staple "$3"
        echo "Stapler done!"
        rm result_tmp
        break
    fi
    if [[ "$i" != "" ]]; then
        echo "Failed!"
        echo "$result"
        rm result_tmp
        return 1
    fi
    echo "Processing, check again after 30 seconds."
    sleep 30
done

{% endhighlight %}

## 參考資料

* [自動化 Notarization 腳本](https://www.logcg.com/archives/3222.html)