Middleware
========
在某些時候，你都會需要讓Django針對每個程式的處理來執行一些程式片段。這些程式碼可能需要在view處理他之前進行一些變動，可能是要針對網站的請求進行除錯需要的目的而做記錄。

你可以透過Django的middleware的框架來達成這些事情，例如在Django的request跟response的物件裡面去處理一些事情。這樣的事情可以作用在Django的輸入或者是輸出的情況，低階的系統外掛方式。

每個middleware元件都被賦與處理某些特定的事情。如果你是按部就班的閱讀本書，你應該已經看過很多次middleware的蹤影了：

所有先前在第十四章看到的session跟使用者工具都是透過一些middleware所達成的。(特別是 middleware讓request.session跟request.user可以讓你在view中直接取得)
在第十五章提到的網站快取的功能，如果目前所請求的頁面已經被快取過了，也是利用middleware來處理。
第十六章的flastpages，redirects，跟csrf應用，所有那些神奇的功能都是透過middleware元件做到的。

這個章節將會深入了解什麼是middleware，以及他是如何辦到的，以及如何自已寫一個你自已的middleware。

什麼是Middleware

就先從一個簡單的範例開始吧。

高流量的網站通常需要將django佈署在具有負載平衡的代理伺服器之下。這可能有一點負雜，其中一項是每一個遠端請求的ip都會是負載平衡器的ip，而不是真正來源的ip。負載平衡器利用 X-Forwarded-For 這個header來處理對方的真實來源ip。

這裡有一個簡單的middleware讓網站可以在proxy後面運作，但是依舊可以透過request.META[“REMOTE_ADDR”] 取得正確的ip位址:

class SetRemoteAddrFromForwardedFor(object):
    def process_request(self, request):
        try:
            real_ip = request.META['HTTP_X_FORWARDED_FOR']
        except KeyError:
            pass
        else:
            # HTTP_X_FORWARDED_FOR can be a comma-separated list of IPs.
            # Take just the first one.
            real_ip = real_ip.split(",")[0]
            request.META['REMOTE_ADDR'] = real_ip
(註:雖然http header 被呼叫了 X-Forwarded-For, Django 讓他可以透過 request.META[“HTTP_X_FORWARDED_FOR”]取得。除了content-length跟 content-type，任何的在request中的header都被包含在request.META裡面，將所有的字元都大寫，並且用底線取代掉連結線，前置字元是 HTTP_)

如果這個middleware被安裝了，每次頁面請求的時候 X-Forwarded-For 的內容就會被自動增加到 request.META[‘REMOTE_ADDR’]。這代表你的Django應用程式不需要去顧慮到是否你的使用者或者是網站處於負載平衡器的代理伺服器後面，他們可以很簡單的透過request.META[‘REMOTE_ADDR’]被存取到，而且不管是否有個代理伺服器正在使用當中。

事實上這個普遍會用到middleware功能是Django內建的。他就位於 django.middleware.http當中，而且你可以在本章節後面看到更多關於他的內容。

安裝 Middleware

如果你是按部就班的閱讀本書，你已經看到很多middleware的安裝範例，在前面很多章節已經安裝過一些middleware。這裡就教你如何安裝middleware。

為了要啟動某一個middleware的元件，在你的設定檔案中的MIDDLEWARE_CLASSES的tuple中新增。在MIDDLEWARE_CLASSES中，每一個middleware元件都是用字串來表示：該middleware的類別完整的python路徑。舉例來說，這裡有一個使用django-admin.py startproject 產生的預設MIDDLEWARE_CLASSES：

MIDDLEWARE_CLASSES = (
    'django.middleware.common.CommonMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
)

一個Django的安裝不需要任何的middleware－MIDDLEWARE_CLASSES可以是空白的，如果你想的話，但是我們建議你要啟動 CommonMiddleware，我們稍後會提到。

安裝的Middleware的排序是有意義的。在request跟view的情況之下，Django會從MIDDLEWARE_CLASSES中依序的將middleware的功能套用上去。而當在response跟exception的情況，Django則是從下到上反向的將middleware套用上去。
Django將MIDDLEWARE_CLASSES的功能包覆在view之外，所以當request到達view的時候會從上到下處理，而response的時候則是由下到上回去。

Middleware的方法
現在你已經知道middleware是什麼跟他如何運作的，讓我們看一下有哪些方法可以在middleware class定義。

Initializer：__init__(self)
使用 __init__() 來對給予的middleware類別進行執行系統的設定。

基於效能的考量，每個啟動的middleware類別只會在每次伺服器處理的時候被實體化。這代表 __init__() 只有一次，當伺服器啟動的時候，而不是每次的單獨網站request出現的時候。

普遍的理由來實作 __init__() 是為了檢查middleware是否被需要使用到，如果 __init__()引發了 django.core.exceptions.MiddlewareNotUsed，然後Django就會將該middleware從middleware的堆疊裡移除。你也許可以使用這個特性來檢驗一些軟體所需要的middleware是否存在，或者是檢查伺服器是否正在除錯的模式，或者是其他類似的情況。

如果一個middleware類別定義了__init__()，這個方法除了標準的self參數之外，不會有其他的參數出現。

Request Preprocessor：process_request(self, request)

這個方法當request被接受的時候就會馬上被呼叫到，甚至在Django去處理URL決定哪個view要被執行之前。他會傳遞你可能會需要去做一些處理的HttpRequest物件。

process_request() 應該會回傳一個 HttpResponse 物件 或者是 None。
如果他回傳None，Django將會繼續處理這個request，執行任何其他middleware跟適當的view。
如果他回傳一個HttpResponse物件，Django將不會試著去呼叫其他的middleware或者是相關的view。Django將會馬上回傳HttpResponse的物件。

View Preprocessor：process_view(self, request, view, args, kwargs)

當Request Preprocessor已經被呼叫，而且Django已經決定哪個view要被執行，但是實際上view尚未被執行之前，會先呼叫view preprocessor。

Table 17-1. Arguments Passed to process_view()
Argument  Explanation
request	HttpRequest物件
view	Django將會呼叫來處理這個request的Python函式。本身將會是一個真正的函式物件，不是那個函式的字串名稱而已。
args	一串參數將會傳遞給view處理的變數，但是不包含request本身(request本身就會是第一個傳遞給view的變數了)
kwargs	會以關鍵字類型參數的一個dictionary來傳遞給view。


如同 process_request()，process_view() 應該回傳 None跟 HttpResponse物件。
如果他回傳None，Django將會繼續處理這個request，執行任何其他的middleware跟相關的view。
如果他回傳 HttpResponse物件，Django將不會呼叫其他任何的middleware跟view。Djago將會馬上回傳該 HttpResponse物件。

Response Postprocessor：process_response(self, request, response)
這個方法在該view已經被呼叫了，而且response已經被產生的時候會被呼叫。這個processor可以變更response的內容。一個明顯的使用例子就是資訊壓縮，例如將request的HTML資訊用gzipping壓縮起來。

Exception Postprocessor：process_exception(self, request, exception)

這個方法只有當如果一個view引發了一個無法處理的例外的時候會被呼叫。你可以使用這個來預先發送錯誤通知，將訊息輸出到log檔案中，或者甚至是將錯誤的情況自動復原。

這個函式的參數跟先前處理的 request 物件一樣，而且exception 事實上他也是一個 載view函式當中會被呼叫的 Exception 物件。

process_exception() 應該回傳一個 HttpResponse物件 或者是 None。
如果他回傳None，Django將會利用目前內建的例外處理繼續處理這個request。
如果他回傳一個 HttpResponse 物件，Django將會使用目前的response結果來處理本身內建的例外處理程序。

Note
Django ships with a number of middleware classes (discussed in the following section) that make good examples. Reading the code for them should give you a good feel for the power of middleware.
You can also find a number of community-contributed examples on Django’s wiki: http://code.djangoproject.com/wiki/ContributedMiddleware

內建的 Middleware

Django內建一些middleware來處理一些普遍的問題，我們會在下面的章節去討論。

Authentication Support Middleware
Middleware類別位置：django.contrib.auth.middleware.AuthenticationMiddleware。
這個middleware會開啟認證支援的功能。他會增加 request.user的屬性，顯示目前登入的使用者的資訊到每個之後的 HttpRequest 物件。

參閱14章節獲得更多的資訊。

“Common” Middleware
Middleware類別的位置：django.middleware.common.CommonMiddleware。

這個middleware新增一些便利性
禁止存在 DISALLOWED_USER_AGENTS設定裡面的user agent存取頁面。如果提供了，這個設定應該是一個利用正規表示式編譯過的物件列表，只要針對近來的request是符合的user-agent。例如：

import re

DISALLOWED_USER_AGENTS = (
    re.compile(r'^OmniExplorer_Bot'),
    re.compile(r'^Googlebot')
)


記得要import re，因為DISALLOWED_USER_AGENTS必須要將他的數值利用正規表示式來呈現。(例如, re.compile())。這個設定檔案是一個標準的Python程式，所以import的語法是合法的。

基於APPEND_SLASH跟PREPEND_WWW設定檔案來執行URL覆寫：
如果APPEND_SLASH是True，URL如果缺少斜線，將會被重導到具有斜線的頁面，除非網址最後一部分包含了時間。所以foo.com/bar會被導到foo.com/bar/，但是foo.com/bar/file.txt不會被做任何變更。

如我PREPEND_WWW是True，URL如果缺少WWW將會被導到相同具有WWW的網址。

這些選項都是提供給正常的網址使用。他的哲學思維是，每一個網址應該存在一個，只有為一個地方。技術上來說，example.com/bar跟example.com/bar/是不同的，即使是www.example.com/bar也是不同的。建立索引的搜尋引擎會將這些網址視為不同的，這會影響到決定你的網站的搜尋引擎排名，所以最好是試著實作這樣的正規化網址。

根據USE_ETAGS設定來處理ETags。ETags是一個HTTP階層的頁面快取最佳化的方式。如果使用 USE_ETAGS為True，Django將會利用MD5加密頁面的內容後，計算一個ETag給每個request，在適當的時機他會傳送回沒有變更的回應。

注意，稍微提到一下，這裡有一個關鍵的 GET middleware，他負責處理ETags以及做更多的事。

Compression Middleware

Middleware類別：django.middleware.gzip.GZipMiddleware。
這個middleware會自動將內容壓縮之後傳送給可以處理gzip壓縮技術的瀏覽器(目前傳統的瀏覽器幾乎都支援！)。這可以大大減少網頁伺服器的流量消耗的情形。他的代價是為了壓縮資訊，所以會需要多一點處理時間。

我們通常比較偏好速度快一點更剩餘頻寬，但是如果你剛好相反，只要將這個middleware啟動就可以了。

Conditional Get Middleware

Middleware類別：django.middleware.http.ConditionalGetMiddleware。

這個middleware提供支援選擇性的GET操作。如果response具有一個Last-Modified或者是ETags或者是標頭，而且request具備 If-None-Match 或者是 If-Modified-Since ，回應就會是一個304(尚未變動)的回應。ETags根據 USE_ETAGS的設定來決定是否支援，及時ETags回應已經被設定。如同上面提到，ETags的檔頭是由Common Middleware來賦予的。

他也會從所有的回應給HEAD的請求中移除內容，而且設定日期跟內容長度給所有的request。

Revers Proxy Support（X-Forwared-For Middleware）
Middleware類別：django.middleware.http.SetRemoteAddrFromForwardedFor。

這是我們剛剛在 “什麼是Middlware” 解釋過。他會根據 request.META[“HTTP_X_FORWARDED_FOR”] 來設定 request.META[“REMOTE_ADDR”] 的數值，如果 HTTP_X_FORWARDED_FOR有設定的話。這是非常有用的，如果你的網站位於一台 reveres proxy 之後，每個request的REMOTE_ADDR位被設為 127.0.0.1。
Danger!
This middleware does not validate HTTP_X_FORWARDED_FOR.
If you’re not behind a reverse proxy that sets HTTP_X_FORWARDED_FOR automatically, do not use this middleware. Anybody can spoof the value of HTTP_X_FORWARDED_FOR, and because this sets REMOTE_ADDR based on HTTP_X_FORWARDED_FOR, that means anybody can fake his IP address.
Only use this middleware when you can absolutely trust the value of HTTP_X_FORWARDED_FOR.

Session Support Middleware

Middleware類別：django.contrib.sessions.middleware.SessionMiddleware。
這個middleware提供 Session的支援，參閱14章節。

Sitewide Cache Middleware

Middleware類別：django.middleware.cache.UpdateCacheMiddleware 跟 django.middleware.cache.FetchFromCacheMiddleware。

這些middleware可以在每個透過Django建立的頁面產生cache。這會在15章討論到。

Transaction Middleware

Middleware類別：django.middleware.transaction.TransactionMiddleware。

這個middleware 對request/response的處理情形 整合了 資料庫的 COMMIT 跟 ROLLBACK 的功能。如果一個view執行成功，就會執行 COMMIT。如果view引發一個exception， ROLLBACK就會被觸發。

Middleware的順序是很重要的。Middleware模組運作是由外而內的，當送出之後就會存檔，跟預設的Django的行為一樣。Middleware

The order of this middleware in the stack is important. Middleware modules running outside of it run with commit-on-save — the default Django behavior. Middleware modules running inside it (coming later in the stack) will be under the same transaction control as the view functions.
參閱附錄B來獲得更多關於資料庫交易的資訊。

下一步是？
網站開發人員跟資料庫規劃設計師不需要總是從頭開始。在下一章節中，我們會知道如何從舊有的系統當中整合，例如從1980年代開始的資料庫的結構。
