表單
========
HTML表單是一個互動式網站的骨幹，從Google單一搜尋欄位的簡潔設計，到各個部落格發送留言的表單，各種自行定義的複雜資料輸入的介面上。這個章節將含括可以如何利用 Django 來處理使用者送出的表單資訊，驗證，然後對他進行處理。除了這個之外我們也會提到 HttpRequest 跟 Form 物件。

從 Request 物件取得資料
我們已經在第三章，第一次探討到 view 的功能時，介紹 HttpRequest 物件過，但是還沒有花很多時間去了解。回想一下每一個 view 的函式都會帶有一個 HttpRequest 的物件當做他第一個參數，例如我們的 hello() view：

from django.http import HttpResponse

def hello(request):
    return HttpResponse("Hello world")

HttpRequest 物件，也就這個範例中的 request，他具備一些特別的屬性跟方法，你應該自己熟悉他們，而且你應該辦的到的。你可以在 view 函式被執行的時候，使用這些屬性來取得目前 request 的資訊（例如：使用者/瀏覽器正在你的Django網站上面載入的資訊）。

Information About the URL

HttpRequest 物件包含許多關於目前被請求網址的資訊：

Attribute/method  Description	Example
request.path	完整的路徑，但是不包含網域，開始會有一個反斜線.	"/hello/"
request.get_host()	網域位址（例如：通常來說就是網域）	"127.0.0.1:8000" or "www.example.com"
request.get_full_path()	完整的路徑，如果有查詢字串的話，會加上查詢的字串。	"/hello/?print=true"
request.is_secure()	如果是透過HTTPS的請求，會傳回True，反之為False。	True or False


多加利用這些屬性跟方法來取代將網址寫死在你的 view 的程式碼當中。這可以讓程式碼比較有彈性，而且可以重複使用在別的地方。一個簡單的例子：

# 不好的!
def current_url_view_bad(request):
    return HttpResponse("Welcome to the page at /current/")

# 好
def current_url_view_good(request):
    return HttpResponse("Welcome to the page at %s" % request.path)

其他關於 Request 的資訊
request.META 是一個python的dictionary，包含所有關於HTTP 標頭相關的訊息－包含使用者的IP位址跟使用者瀏覽器的資訊。需要注意的是，所有完整的標頭資訊是根據使用者所送出的訊息跟你網站伺服器所設定。一些普遍可以取得的關鍵資訊例如：
HTTP_REFERER－參考來源網址，如果有的話。（注意不要拼錯 REFERER）
HTTP_USER_AGENT－使用者瀏覽器的名稱，可以是任何名稱。會看起來像是："Mozilla/5.0 (X11; U; Linux i686; fr-FR; rv:1.8.1.17) Gecko/20080829 Firefox/2.0.0.17"。
REMOTE_ADDR－客戶端的IP來源，例如 “12.345.67.89”。（如果請求有透過代理伺服器的方式來傳遞，那可以能會是一個利用逗號來分割的列表，例如：”12.345.67.89,23.456.78.90”。）

注意，因為 request.META 是一個基本的Python dictionary，如果你試著去存取一個不存在的 key，你會得到一個 KeyError 的例外訊息。(因為 HTTP 標頭是外部的資訊－他們是透過瀏覽器送出的－也就是說他們的資訊並不一定是可信賴的，所以你應該設計避免讓程式因為碰到資訊不存在或者是空的時候的錯誤。）你應該使用 try/except 的語法或者是 get()來處理一些尚未被定義的 key：
# 不對的!
def ua_display_bad(request):
    ua = request.META['HTTP_USER_AGENT']  # Might raise KeyError!
    return HttpResponse("Your browser is %s" % ua)

# 正確的 (版本 1)
def ua_display_good1(request):
    try:
        ua = request.META['HTTP_USER_AGENT']
    except KeyError:
        ua = 'unknown'
    return HttpResponse("Your browser is %s" % ua)

# 正確的 (版本 2)
def ua_display_good2(request):
    ua = request.META.get('HTTP_USER_AGENT', 'unknown')
    return HttpResponse("Your browser is %s" % ua)

我們鼓勵你寫一個小的 view 來顯示所有的 request.META 的資訊，你可以知道有哪些東西在裡面。這是那個 view 可能會長得樣子：

def display_meta(request):
    values = request.META.items()
    values.sort()
    html = []
    for k, v in values:
        html.append('<tr><td>%s</td><td>%s</td></tr>' % (k, v))
    return HttpResponse('<table>%s</table>' % '\n'.join(html))

試著將這個 view 利用Django樣板系統取代原先寫死的HTML。也可以順便試試看增加前面章節學到的 request.path 跟其他 HttpRequest 行為。

關於表單送出的資訊
除了關於 request 的基本資訊之外，HttpRequest 物件也具備兩個由使用者所送出資料的屬性：request.GET 跟 request.POST。這兩個都是類似 dictionary 的物件，可以讓你存取 GET 跟 POST 傳遞出來的資訊。

像Dictionary的物件
當我們提到 request.GET 跟 request.POST 是 “像Dictionary的物件”，意思也就是說他們的行為跟標準的Python dictionary一樣，但是技術上來說，dictionary是比較底層的。例如：request.GET 跟 request.POST 都具備 get(), keys() 跟 values()的方法，而且你可以利用for key in request.GET 的方式來將 key 的資料列舉出來。
所以他們兩者的差異是？因為 request.GET跟 request.POST 具備一般dictionary物件所沒有的額外方法。等一下會看到這部份。

你可能也會碰到一些類似的情況，例如 “像檔案的物件”－Python物件具備一些行為，例如 read()，讓他們可以跟真的檔案物件一樣進行操作。


POST 資訊一般來說是透過HTML的 <form> 送出，而GET 資訊可以來自 <form> 或者是URL的字串查詢。

一個簡單的表單處理範例
繼續這本書的範例，作者跟發行商，讓我們建立一個簡單的 view 來讓使用者透過標題來搜尋我們的書籍資料庫。

一般來說，會有兩個部分需要製作表單：一個是HTML的使用者介面跟後端 view 程式碼來處理送出的資訊。第一個必份很簡單，我們只要設定 view 來顯示一個搜尋的表單：
from django.shortcuts import render_to_response

def search_form(request):
    return render_to_response('search_form.html')

如我們在第三章學到的，這個 view 可以放在你的 Python 路徑的任何地方。以參數的角度來說，會放在 books/views.py。

完成的樣板 search_form.html ，可能會長得像這樣：

<html>
<head>
    <title>Search</title>
</head>
<body>
    <form action="/search/" method="get">
        <input type="text" name="q">
        <input type="submit" value="Search">
    </form>
</body>
</html>

在 urls.py 中的URLpattern 應該會是這樣：
from mysite.books import views

urlpatterns = patterns('',
    # ...
    (r'^search-form/$', views.search_form),
    # ...
)

（注意，我們直接匯入了 views 的模組，而不是 from mysite.views import search_form，因為前者比較簡潔一點。我們會在第八章提到更多關於匯入方法的細節。）

現在，如果你執行 runserver，接著去瀏覽 http://127.0.0.1:8000/search-form/，你將會看到搜尋的介面。簡單吧。

試著將輸入的資訊送出表單，你會收到Django的404錯誤訊息。因為這個表單指向 /search/，但是他尚未被建立。讓我們利用第二個 view 來修正：
# urls.py

urlpatterns = patterns('',
    # ...
    (r'^search-form/$', views.search_form),
    (r'^search/$', views.search),
    # ...
)

# views.py

def search(request):
    if 'q' in request.GET:
        message = 'You searched for: %r' % request.GET['q']
    else:
        message = 'You submitted an empty form.'
    return HttpResponse(message)

這個時候，應該可以顯示搜尋的結果了，所以我們可以確定資料已經被傳送到Django了，而且你可以大概明白搜尋表單的運作流程。簡單的說：
HTML中的 <form> 定義一個變數q。當他被送出的時候，q會利用 GET (method=”get”）的方式傳送到 /search/。
Django 的 view 會處理 /search/ （search()）然後在 request.GET 中去存取到 q 的數值。

一個重要的事情在這邊要被提醒的是，我們要檢查 q 是否存在 request.GET中。就如同在 request.META 中提到，你不應該相信任何來自使用者送出的訊息或者是他們已經在先前送出過的任何資訊。如果我們沒有進行這樣的確認，任何送出空白資訊的表單都會引起一個 KeyError的錯誤：

# 錯誤的!
def bad_search(request):
    # 接下來的程式將會引起一個 KeyError 的例外，如果 q 沒有資料
    # 被送出。
    message = 'You searched for: %r' % request.GET['q']
    return HttpResponse(message)


查詢字串參數
因為 GET 的資訊是透過查詢字串的方式傳遞 (e.g., /search/?q=django), 你可以使用 request.GET 來存取查詢字串的變數。在第三章介紹 Django的 URLconf 系統的時候我們有比較 Django 漂亮的 URL格式跟傳統的 PHP/Java 網址，如 /time/plus?hours=3 ，而且提到在第七章將會告訴你如何辦到。現在你已經知道如何在你的 view 當中存取查詢字串的參數資訊了(例如 hours = 3 在這個範例中)，使用 request.GET。

POST 資訊的運作方式跟 GET 的資訊一樣－只是使用 request.POST 取代 request.GET。那 GET 跟 POST 的差別在哪邊呢？當送出表單是使用 request.get 的行為的時候使用 GET 。無論何時表單送出後會產生邊際效應－例如，修改資訊或者是寄發一封信件，或者是其他除顯示資訊以外的行為，都使用 POST 。在我們書籍搜尋系統當中，我們使用 GET ，因為這個查詢不會更改任何主機上面的資訊。（請參閱 http://www.w3.org/2001/tag/doc/whenToUseGet.html ，如果想要知道更多關於 GET 跟 POST 的相關資訊。）

現在我們已經確定 request.GET 已經被正常傳遞過去，let’s hook 使用者的查詢到我們的書籍資料庫中（一樣是在 views.py）：


from django.http import HttpResponse
from django.shortcuts import render_to_response
from mysite.books.models import Book

def search(request):
    if 'q' in request.GET and request.GET['q']:
        q = request.GET['q']
        books = Book.objects.filter(title__icontains=q)
        return render_to_response('search_results.html',
            {'books': books, 'query': q})
    else:
        return HttpResponse('Please submit a search term.')


一些關於這邊處理的事情的說明：
檢查 ‘q’ 是否存在 request.GET，我們也確定了 request.GET[‘q’] 是一個不是空的數值，在傳遞到資料庫去做查詢之前。
我們正在使用 Book.objects.filter(title__icontains=q) 來查詢我們的書籍表格裡面全部的書標題包含了查詢的關鍵字。icontains 是一個查詢類別（在第五章跟附錄B裡面會解釋到），而且該描述可以簡單的說明是當成 取得書籍中標題包含了 q，而且不分大小寫。
   這是很簡單的方式去做書籍查詢。我們不建議使用簡單的 icontains 的查詢在大量的線上資料庫系統當中，因為他會很慢。（在現實世界當中，你會想要使用一些客製化搜尋系統。在網路上面搜尋 open-source full-text search 來找尋可能的方法。
我們傳遞了 books，一個 Book 物件的列表到樣板。這個 search_results.html 樣板的程式碼可能會包含某些這樣的東西：
<p>You searched for: <strong>{{ query }}</strong></p>

{% if books %}
    <p>Found {{ books|length }} book{{ books|pluralize }}.</p>
    <ul>
        {% for book in books %}
        <li>{{ book.title }}</li>
        {% endfor %}
    </ul>
{% else %}
    <p>No books matched your search criteria.</p>
{% endif %}

注意 pluralize 樣板過濾器的使用方式，會根據找尋到的書籍數量，會產生一個 s 。

改進我們簡單的表單處理範例
在前面的章節中，我們已經展示給你看到可以運作的簡單例子。現在我們將會指出一些問題，並且告訴你如何改進他。

首先，我們的 search() 的 view 處理一個空的查詢是很弱的－我們要顯示一個 “請輸入一個搜尋字串” 訊息，要求使用者去點擊瀏覽器的返回按鈕。這是一件很糟糕而且不專業的，而且如果你曾經實際建置像這樣的情況，你 Django 的職責將會被撤銷。

最好是重新顯示這個表單，包含一個錯誤的訊息在上面，所以使用者可以直接馬上在試一次。最簡單的方法就是直接重新顯示樣板，像是這樣：

from django.http import HttpResponse
from django.shortcuts import render_to_response
from mysite.books.models import Book

def search_form(request):
    return render_to_response('search_form.html')

def search(request):
    if 'q' in request.GET and request.GET['q']:
        q = request.GET['q']
        books = Book.objects.filter(title__icontains=q)
        return render_to_response('search_results.html',
            {'books': books, 'query': q})
    else:
        return render_to_response('search_form.html', {'error': True})

（注意，我們已經包含了一個 search_form() 所以你可以在一個地方看到兩個views）
我們已經改進了 search() 來在一次顯示 search_form.html 樣板，如果查詢是空的。然後，因為我們需要去顯示一個錯誤訊息在樣板當中，我們會傳遞一個樣板變數。現在我們可以編輯 search_form.html 來檢查 error 變數：

<html>
<head>
    <title>Search</title>
</head>
<body>
    {% if error %}
        <p style="color: red;">Please submit a search term.</p>
    {% endif %}
    <form action="/search/" method="get">
        <input type="text" name="q">
        <input type="submit" value="Search">
    </form>
</body>
</html>

我們可以依舊從我們原來的 view 當中使用這個樣板，search_form()，因為 search_form() 不會傳遞 error 給樣板－所以錯誤的訊息將不會顯示在這個情況。

有了這個地方的改進，這個應用程式變得比較好了，但是他現在找出另外一個問題：是否一個 search_form() 的 view 是必須的呢？當他存在的時候，一個請求給 /search/ （沒有任何的 GET 查詢參數）將會顯示空白的表單（但是會有一個錯誤）。我們可以移除掉 search_form() 的 view，包含相關的網址設定，當有人瀏覽 /search/ 的頁面沒有任何 GET 的參數時，隱藏錯誤的訊息。

def search(request):
    error = False
    if 'q' in request.GET:
        q = request.GET['q']
        if not q:
            error = True
        else:
            books = Book.objects.filter(title__icontains=q)
            return render_to_response('search_results.html',
                {'books': books, 'query': q})
    return render_to_response('search_form.html',
        {'error': error})


在這個更新的 view ，如果一個使用者瀏覽了 /search/ 沒有包含任何的 GET 參數，他將會看到搜尋的表單，不會有任何的錯誤訊息。如果一個使用者送出了不含任何給 q 的查詢字樣的表單，他將會看到搜尋表單會包含一個錯誤訊息。最後，如果一個使用者送出表單並且具備不是空的 q 的數值，他將會看到搜尋結果。

我們可以製作一個最終改進過的版本，移除掉一些重複性的程式碼。現在，我們已經翻修了兩個 view 跟網址成為一個。而且 /search/ 處理了表單的顯示跟搜尋結果的呈現，search_form.html 當中的 <form> 也不需要將網址固定寫在裡面。改成這樣的寫法：
<form action="/search/" method="get">

他可以改成這樣
<form action="" method="get">

action=”” 代表 “送出這個表單給目前頁面相同的網址”。有了這樣的改變，你將不需要記得去修改 action ，如果你想要將 search() 的 view 改到其他的網址。

簡單的驗證
我們的搜尋範例非常的簡單，特別是在他資料驗證的地方；我們

