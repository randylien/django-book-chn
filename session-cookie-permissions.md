Session, Cookie, Permissions
========
Cookie

在HttpRequest的物件裡具備了Cookie的物件，只要跟操作dictionary一樣的方式就可以取得cookie的資訊。

return request.COOKIES['username']

就是這麼簡單

如果要設定cookie的資訊，則是要透過HttpResponse的物件來操作，並且使用它內建的set_cookie的方法。

return response.set_cookie("username", "randylien")

但是Cookie本身並不夠安全。


Django's session framework

啟動django 的session
在MIDDLEWARE_CLASSES加入
django.contrib.session.middleware.SessionMiddleware
還有在INSTALL_APPS加入
django.contrib.session

如果你新增了session的功能，記得要manage.py syncdb讓他建立相關的資料表

啟動session之後，我們的HttpRequest物件就會多了一個session的物件
所以你就可以跟存取dictionary一樣的存取session裡面的資訊

request.session['username'] = "randylien"

print request.session['username']
del request.session['username']

有幾點在使用session需要注意的

命名的時候用底線開頭的key預設是給django用的，所以命名的時候不要使用底線開頭
請不要取代掉request.session的物件或者是對request.session物件增加屬性。

Session & Cookie

如同之前提到Cookie的問題，我們要怎麼知道使用者的瀏覽器可不可以接受Cookie呢？django的session framework提供了一個set_test_cookie()的方法可以讓我們測試。

可以透過 request.session.test_cookie_worked() 檢查測試的結果。

如果成功可以使用 request.delete_test_cookie() 將測試過的結果刪除掉。

PS.內建的認證系統都幫你將上述的事情都做好了。


在View以外的地方使用Session

簡單來說，session其實就是個一般的django model，利用32bit的隨機字元存取起來在你的cookie當中，所以你可以透過這組字元進行查詢某一組session的實體，並且取得裡面的資訊。
不過session_data本身都有經過加密，所以你還需要呼叫get_decoded()來將資料還原。

Session的資訊何時會被django存起來呢？

當request.session變動的時候。

request.session['username']='randy'

del request.session['username']

request.session['username']={}



但是

request.session['username']['lastname']='lien'

並不會進行session的存檔行為。

如果要讓django每次都存session的資訊，必須要修改 SESSION_SAVE_EVERY_REQUEST ，將它設定為True。這樣一來每次對session存取的時候都會去存最後的session資訊。
但是要注意的是，如此一來cookie的expire就會每次都變動到。


SESSION_EXPIRE_AT_BROWSER_CLOSE 預設是False，換句話說使用者的session的存活時間時間內(預設是1,209,600秒)，它下一次打開瀏覽器的時候，依舊可以取得先前的狀態。
透過這樣的方式可以達到不需要每次都登入的效果。

如果設定為True，django將會使用瀏覽器長度的cookies。

其他的Session設定

SESSION_COOKIE_DOMAIN  
設定session的網域區域，如果需要跨網域的cookie的時候，就設定為 ".randy.com"，預設為None

SESSION_COOKIE_NAME
session在cookie中的id

SESSION_COOKIE_SECURE
是否要使用https來傳輸cookie

更多關於session的技術細節
session的dictionary接受任何被pickle處理過的物件資訊。
session是被存在django_session的資料表格裡面。
session只有當被呼叫到request.session的時候才會去抓取資料庫。
django只有當session的資料有被變動的時候才會送出，除非將SESSION_SAVE_EVERY_REQUEST設定為True。
Django的session framework是很完整，原創基於cookie的。它不會傳會任何的session id在網址列上面。
如果你對於session的實作有興趣，可以直接看看django.contrib.sessions的原始碼獲得更多的啟示。

使用者與認證系統


Django提供工具讓你去處理這些普遍的工作事項。Django的使用者認證系統處理使用者的帳戶，群組，權限跟cookie基礎的使用者session。
這樣的系統通常視為是auth/auth 認證跟驗證系統。
辨識使用者通常需要兩個步驟，我們需要
驗證，這個使用者是誰(通常使用帳號密碼的認證方式)
辨識使用者的權限。
基於這樣的需求，Django的auth/auth系統由幾個部分構成，

使用者：人們註冊你的網站。
權限：決定使用者是否有辦法進行某些工作。
群組：普遍用來套用權限跟標籤到多人的情況。
訊息：簡單的處理佇列跟系統資訊給使用者的方式。


開啟認證支援

如同session工具，認證系統也被包含在django的應用當中。跟session一樣也是預設就被安裝好了，但是如果你已經移除掉它，你需要

確定session framework已經被安裝好了(參照前面的章節)，確保使用者的cookie可以被存取。
在INSTALL_APPS中增加 django.contrib.auth ，然後執行 manage.py syncdb 讓他建立所需要的資料庫欄位。
確定 django.contrib.auth.middleware.AuthenticationMiddleware 已經在你的 MIDDLEWARE_CLASSES 設定中，並且要在 SessionMiddleware之後。
安裝好之後，就可以對user做處理了。你可以透過request.user的介面來存取資訊，這個物件會代表目前登入的使用者資訊，如果使用者尚未登入，這個使用者將會是個 AnonymousUser 的物件。

你可以透過 is_authenticated() 的方法簡單的去知道使用者是否有登入。

if request.user.is_authenticated():
	#do 
else:
	#do

使用使用者

只要你有個User，通常來自request.user。
但是也有可能透過別的方式取得，在這個物件裡面有很多欄位跟method可以使用。
AnonymousUser 物件也遵循這個行為，但是不是全部，所以在操作user物件的之前，你可能要常常進行 is_authenticated的檢查。

下面表格列出來User 這個欄位，行為。



Table 14-3. Fields on User Objects
Field	Description
username	必需的。30個字元以下，只允取英文字母跟數字還有底線。
first_name	選擇性，30個字元以下
last_name	選擇性，30個字元以下
email	選擇性，電子郵件的格式
password	必需的。經過加密後的資訊(Django不存原始的密碼)。可以參考『密碼』以獲得更多關於這個欄位的資訊。
is_staff	布林值。是否這個使用者可以存取管理人員的網站。
is_active	布林值。是否這個帳戶可以被使用並且登入。設定 False 來代替刪除帳號。
is_superuser	布林值。是否這個帳戶具有所有管理的權限。
last_login	使用者最後一次登入的時間。預設是當時的日期跟時間。
date_joined	使用者建立的時間。預設是帳戶建立的時間。
1
Table 14-4. Methods on User Objects
Method	Description
is_authenticated()	回傳 True 的時候代表這是一個 User 物件。這是知道使用者是否已經通過認證的方法。不會知道使用者的權限關係跟帳戶是否啟動。他只會告訴你這個使用者是否經過認證。
is_anonymous()	回傳 True 的時候代表這是一個 AnonymousUser 物件(回傳False 代表 是一個 User物件)。一般來說會使用 is_authenticated()。
get_full_name()	回傳 first_name 加上 last_name 的字串，中間會有空白。
set_password(passwd)	透過提供的字串來設定使用者的密碼，這個動作不會真正將 User 物件存入。
check_password(passwd)	如果提供的字串是開使用者的密碼就會回傳True，這個函式會對經過加密的字串進行比對。
get_group_permissions()	回傳該使用者所屬的群組中所具備權限的字串。
get_all_permissions()	回傳該使用者所屬的群組與本身具備權限的字串。
has_perm(perm)	如果該使用者具備特定的權限，將會回傳 True。perm的格式為 "package.codename"。如果該使用者尚未啟動，則只會回傳 False。
has_perms(perm_list)	如果該使用者具備所有perm_list權限，將會回傳 True。如果該使用者尚未啟動，則只會回傳 False。
has_module_perms(app_label)	如果該使用者具備任何符合 app_label的權限，將會回傳 True。如果該使用者尚未啟動，則只會回傳 False。
get_and_delete_messages()	Returns a list of Message objects in the user’s queue and deletes the messages from the queue.
email_user(subj, msg)	發送信件給使用者。該信件會根據 DEFAULT_FROM_MAIL 的設定來發送。你可以透過提供額外的第三個參數 from_email 來取代。

最後，User物件有兩個ManyToMany的欄位：groups 跟 permissions。所以從User物件可以利用many-to-many的欄位去存取他們相關的物件。

# Set a user's groups:
myuser.groups = group_list

# Add a user to some groups:
myuser.groups.add(group1, group2,...)

# Remove a user from some groups:
myuser.groups.remove(group1, group2,...)

# Remove a user from all groups:
myuser.groups.clear()

# Permissions work the same way
myuser.permissions = permission_list
myuser.permissions.add(permission1, permission2, ...)
myuser.permissions.remove(permission1, permission2, ...)
myuser.permissions.clear()

登入與登出

Django提供內建的方法來處理登入與登出的問題(還有一些其他的小技巧)。但是在我們使用這些之前，先來了解如何自己處理這些事情。Django提供兩個函式去處理這些行為：django.contrib.auth.authenticate 跟 login()。

為了要認證一個使用者跟密碼，使用 authenticate()。需要兩個關鍵的參數，username跟password，如果提供的帳號密碼被驗證成功後，他會回傳一個User的物件。如果失敗 authenticate()會回傳一個None的物件。

>>> from django.contrib import auth
>>> user = auth.authenticate(username='john', password='secret')
>>> if user is not None:
...     print "Correct!"
... else:
...     print "Invalid password."

authenticate() 只會驗證使用者。如果要登入某個使用者則是要使用login()。他需要一個HttpRequest 跟 User的物件透過Django的session framework來將該使用者的id存入session當中。

這個範例告訴你如何利用 authenticate() 跟 login() 在一個view function中。

from django.contrib import auth

def login_view(request):
    username = request.POST.get('username', '')
    password = request.POST.get('password', '')
    user = auth.authenticate(username=username, password=password)
    if user is not None and user.is_active:
        # Correct password, and the user is marked "active"
        auth.login(request, user)
        # Redirect to a success page.
        return HttpResponseRedirect("/account/loggedin/")
    else:
        # Show an error page
        return HttpResponseRedirect("/account/invalid/")
為了登出一個使用者，使用 django.contrib.auth.logout()。他需要透過一個 HttpRequest 物件而且不會有任何回傳值。

from django.contrib import auth

def logout_view(request):
    auth.logout(request)
    # Redirect to a success page.
    return HttpResponseRedirect("/account/loggedout/")
注意，如果使用者沒有登入， auth.logout() 不會回傳任何的錯誤。

在實作的時候，你不需要自己寫 登入跟登出的函式。認證系統都具備了這方面的行為處理的功能。使用認證系統的第一步驟就是設定URLconf。你需要增加下列的程式片段：

from django.contrib.auth.views import login, logout

urlpatterns = patterns('',
    # existing patterns here...
    (r'^accounts/login/$',  login),
    (r'^accounts/logout/$', logout),
)
所以 /accounts/login/ 跟 /accounts/logout/ 將會是預設的URL給Django來處理的這些view。
/accounts/login/ and /accounts/logout/ are the default URLs that Django uses for these views.


login view會利用 registration/login.html的樣板來產生頁面(你可以透過提供額外的參數來改變樣板的名稱, template_name)。這個表格需要包含一個username跟password的欄位，一個簡單的樣板也許看起來像這樣：

{% extends "base.html" %}

{% block content %}

  {% if form.errors %}
    <p class="error">Sorry, that's not a valid username or password</p>
  {% endif %}

  <form action="" method="post">
    <label for="username">User name:</label>
    <input type="text" name="username" value="" id="username">
    <label for="password">Password:</label>
    <input type="password" name="password" value="" id="password">

    <input type="submit" value="login" />
    <input type="hidden" name="next" value="{{ next|escape }}" />
  </form>

{% endblock %}
如果使用者成功登入，他將會被轉址到 /accounts/profile/ 。你可以透過提供一個隱藏的欄位叫做 next 來改變登入後轉址的目的。你也可以利用GET來傳遞這個變數，他會自動將變數存到你設定的隱藏欄位中。

登出的作法有點不同。預設是他會利用 registration/logged_out.html 的樣板來產生頁面（通常會包含你已經成功登出的訊息）。然而你可以透過額外傳遞變數 next_page，在登出之後就會自動轉址過去。

限制登入後的使用者存取權限

當然，為了要限制使用者存取我們的網站，我們將要來解決這些問題。
最簡單，原始的方法去限制使用者存取頁面就是透過 request.user.is_authenticated() 跟轉址到登入頁面。

from django.http import HttpResponseRedirect

def my_view(request):
    if not request.user.is_authenticated():
        return HttpResponseRedirect('/accounts/login/?next=%s' % request.path)
    # ...
或者是顯示一個錯誤訊息：

def my_view(request):
    if not request.user.is_authenticated():
        return render_to_response('myapp/login_error.html')
    # ...
更簡單的方式就是利用 login_required 這個裝飾字，
from django.contrib.auth.decorators import login_required

@login_required
def my_view(request):
    # ...
login_required 將會處理下面這些步驟：
如果使用者沒有登入，將會轉址到 /accounts/login/ ，並且傳遞當時的頁面網址為next參數，例如：/accounts/login/?next=/polls/3/。
如果使用者已經登入了，會執行該view。該程式會根據使用者登入的情況去執行。
限制使用者存取
存取限制是基於某些權限或者是其他的檢驗測試，或者是提供不同的頁面給登入的使用者做處理。

最原始的方法是直接將request.user放在你的view當中進行測試。
舉例來說，這個頁面檢查是否該使用者已經登入，而且具備 polls.can_vaote的權限(更多關於全縣的部份之後會提到)：

def vote(request):
    if request.user.is_authenticated() and request.user.has_perm('polls.can_vote')):
        # vote here
    else:
        return HttpResponse("You can't vote in this poll.")
Django提供一個快捷的方法， user_passes_test 裝飾字元。他具備參數並且成為一個特別的裝飾字元讓你在某些情況可以使用：

def user_can_vote(user):
    return user.is_authenticated() and user.has_perm("polls.can_vote")

@user_passes_test(user_can_vote, login_url="/login/")
def vote(request):
    # Code here can assume a logged-in user with the correct permission.
    ...
user_passes_test 需要一個必備的參數：一個可以被呼叫的函式，並且帶有User物件的參數，如果使用者允許檢視這個頁面將會回傳True。注意，user_passes_test 不會自動檢查 User 物件是否被驗證過，你應該要自己處理這部份。

在這個範例當中我們也顯示第二個參數，login_url，讓你可以指定登入的頁面，預設是 /accounts/login/。如果使用者沒有通過那個測試，user_passes_test 裝飾字元將會轉址該使用者到登入頁面。

因為這關係到是否該使用者具備部分的權限。Django提供一個裝飾字元給這樣的情況：permission_required()。使用這個裝飾字元後，剛剛的範例可以改寫成這樣。

from django.contrib.auth.decorators import permission_required

@permission_required('polls.can_vote', login_url="/login/")
def vote(request):
    # ...
限制存取到一般的view
Django使用者最常問的一個問題就是如何限制存取某一個generic view。想要解決這個問題，你將會需要寫一個wrapper將view包起來，然後在URLconf的時候指到該wrapper。

from django.contrib.auth.decorators import login_required
from django.views.generic.date_based import object_detail

@login_required
def limited_object_detail(*args, **kwargs):
    return object_detail(*args, **kwargs)
當然你也可以把login_required取代成其他的限制裝飾字元。

管理使用者，權限，跟群組

到目前為止，最方便的方法去管理認證系統是透過admin介面。第六章已經討論到如何使用Django的管理網站去編輯跟控制他們的權限跟存取。大部分你也只需要利用這個介面。

然後，Django也有提供低階的API，當你需要絕對性的控制時，你可以深入去使用，我們將在後面去探討。

增加使用者
利用 create_user 來新增一個使用者：
>>> from django.contrib.auth.models import User
>>> user = User.objects.create_user(username='john',
...                                 email='jlennon@beatles.com',
...                                 password='glass onion')
在這邊，user 是一個User的實體，已經準備好被存到資料庫當中（create_user()不會自己去呼叫save()）。在呼叫save()之前你可以繼續對這個實體做變更。

user.is_staff = True
user.save()

改變密碼
你可以利用 set_password()來改變使用者的密碼：
>>> user = User.objects.get(username='john')
>>> user.set_password('goo goo goo joob')
>>> user.save()
除非你知道你在做什麼，不然千萬不要直接設定密碼。這個密碼也是經過加密，所以無法被直接編輯。

更正確的說，使用者的password的屬性是一個這樣的字串：

hashtype$salt$hash

這是一個hash型態。

hashtype可以是一個sha1(預設)或者是md5，這個演算法都是對password進行單向處理。
salt是一個隨機的字串，用來對原始密碼進行加密處理用，例如：
sha1$a1976$a36cc8cbf81742a8fb52e221aaeab48ed7f58ab4

User.set_password()跟User.check_password() 這些函式用來處理檢查跟設定密碼的情況。


Salted hashes
A hash is a one-way cryptographic function — that is, you can easily compute the hash of a given value, but it’s nearly impossible to take a hash and reconstruct the original value.
If we stored passwords as plain text, anyone who got their hands on the password database would instantly know everyone’s password. Storing passwords as hashes reduces the value of a compromised database.
However, an attacker with the password database could still run a brute- force attack, hashing millions of passwords and comparing those hashes against the stored values. This takes some time, but less than you might think.
Worse, there are publicly available rainbow tables, or databases of pre-computed hashes of millions of passwords. With a rainbow table, an experienced attacker could break most passwords in seconds.
Adding a salt — basically an initial random value — to the stored hash adds another layer of difficulty to breaking passwords. Because salts differ from password to password, they also prevent the use of a rainbow table, thus forcing attackers to fall back on a brute-force attack, itself made more difficult by the extra entropy added to the hash by the salt.
While salted hashes aren’t absolutely the most secure way of storing passwords, they’re a good middle ground between security and convenience.

處理註冊
我們可以使用這些低階的工具來建立讓使用者註冊建立帳號的動作跟頁面。
不同的開發人員利用不同的方法建立註冊的程序。所以django將這些工作留給你。幸運的是這些工作相當的簡單。

簡單的情況，我們可以提供一個簡單的view告訴使用者必需的資訊然後建立帳號。Django提供內建的表單，在這個情況下你可以直接使用。
from django import forms
from django.contrib.auth.forms import UserCreationForm
from django.http import HttpResponseRedirect
from django.shortcuts import render_to_response

def register(request):
    if request.method == 'POST':
        form = UserCreationForm(request.POST)
        if form.is_valid():
            new_user = form.save()
            return HttpResponseRedirect("/books/")
    else:
        form = UserCreationForm()
    return render_to_response("registration/register.html", {
        'form': form,
    })
這個表單叫做 registration/register.html。這個樣板應該看起來是這樣：
{% extends "base.html" %}

{% block title %}Create an account{% endblock %}

{% block content %}
  <h1>Create an account</h1>

  <form action="" method="post">
      {{ form.as_p }}
      <input type="submit" value="Create the account">
  </form>
{% endblock %}

在樣板當中使用驗證過的資訊
當你使用了RequestContext後，目前已經登入的使用者跟他的權限資訊都可以在template的context中被存取。(第九章)


Note
Technically, these variables are only made available in the template context if you useRequestContext and your TEMPLATE_CONTEXT_PROCESSORS setting contains"django.core.context_processors.auth", which is the default. Again, see Chapter 9 for more information.


當使用了 RequestContext，目前使用者(可能是AnonymousUser或者是User的實體)被存在template的變數{{ user }}當中：

{% if user.is_authenticated %}
  <p>Welcome, {{ user.username }}. Thanks for logging in.</p>
{% else %}
  <p>Welcome, new user. Please log in.</p>
{% endif %}
這個使用者的權限將會被存在樣板變數 {{ perm }}。

這裡有兩個方法你可以使用這個 perm 物件。你可以使用類似 {% if perms.polls %} 去檢查是否該使用者有該應用程式的權限，或者是你可以使用一些類似 {% if perms.polls.can_vote %} 檢查是否使用者具備某種權限。

因此，你可以利用 {% if %} 在樣板當中檢驗權限：

{% if perms.polls %}
  <p>You have permission to do something in the polls app.</p>
  {% if perms.polls.can_vote %}
    <p>You can vote!</p>
  {% endif %}
{% else %}
  <p>You don't have permission to do anything in the polls app.</p>
{% endif %}
權限，群組，跟訊息

這裡有一些其他關於 authentication framework的部份。我們將會在後面的部份更仔細的了解。

權限
權限是一個簡單的方法去標示使用者跟群組可以進行一些行為。通常被Django Admin網站使用。但是你可以簡單的利用你自己的程式碼來處理。
Django Admin網站使用權限的部份
存取該view當中add表單，新增物件的行為是根據該使用者是否有被限制對於該物件進行新增的權限。
處理檢視修改列表，檢查變更的表單，而變更一個物件是根據該使用者是否具有變更該物件的權限。
處理刪除一個物件是根據該使用者是否具有對該物件進行修改的權限。
權限處理是根據物件類型去做設定，不是針對某一個物件實體。舉例來說，"瑪麗可以變更news stories"，但是權限不能讓你這樣做 "瑪麗可以變更news stories，但是只有他自己建立的。" 或者是 "瑪麗只能變更某些情況，發佈時間或者是id的 news stories"。

這三個基本的權限－新增，變更，刪除－自動會被每個Django的Model所建立。
除此之外，當你執行 manage.py syncdb 的時候，這些權限都會被增加到 auth_permission 資料庫當中。

這些權限會是這樣的格式 "<app>.<action>_<object_name>"。如果你有個polls的應用程式，具備一個Choice的model，你會得到的權限會長這樣 "polls.add_choice", "polls,change_choice" 跟 "polls.delete_choice"。

就如同使用者，權限會被 django.contrib.auth.models 的Django model實作。這意味著你可以使用Django的資料庫API直接跟權限做處理。

群組

群組是一個簡單的方法來對使用者做分類，所以你可以直接配置權限或者是其他的標籤給這些使用者。一個使用者可以屬於多個群組。

一個使用者在群組中會自動取得該群組的權限。舉例來說，如果該群組Site editors
 有權限 can_edit_home_page，任何使用者在該群組當中都具備該權限。

群組也是一個方便的方法去分類使用者並且給他們相關的標籤，或者是擴充功能。舉例來說，你可以建立一個群組叫做"Special users"，然後你可以讓這些使用者只存取會員專屬的頁面，或者是發送會員專屬的訊息。

跟使用者一樣，最簡單的管理群組的方法就是透過admin site。然而，群組也是基於django.contrib.auth.models 的 Django model所實作，意味著你也可以利用資料庫的API來處理群組。

訊息
Message 系統是一個輕量化的方法來處理給使用者的訊息。一個訊息會跟一個使用者產生關聯。

Django Admin site介面中完成一個行為之後會發現Message的蹤影。舉例來說，當你新增一個物件，你會注意到 "The object was created successfully" 的訊息在admin頁面的最上方。

你可以使用相同的API去處理跟顯示訊息在你自己的應用程式當中。這個API很簡單：

建立一個新的message，使用 user.message_set.create(message='meesage_text')。
接收/刪除訊息，使用 user.get_and_delete_message()，回傳一串存在該使用者佇列當中的Message物件，並且從該佇列當中刪除。

在例子當中，這個系統在建立一個播放清單之後會存一個訊息給該使用者。
def create_playlist(request, songs):
    # Create the playlist with the given songs.
    # ...
    request.user.message_set.create(
        message="Your playlist was added successfully."
    )
    return render_to_response("playlists/create.html",
        context_instance=RequestContext(request))
當你使用RequestContext，該名登入的使用者就可以在template當中透過 {{ message }} 變數取得他的訊息。例如：
{% if messages %}
<ul>
    {% for message in messages %}
    <li>{{ message }}</li>
    {% endfor %}
</ul>
{% endif %}
注意，RequestContext會呼叫 get_and_delete_message ，即使你沒有顯示message，message都會被刪除掉。

最後，注意message framework只能作用在存在user資料庫當中的使用者。如果要發送message給匿名的使用者，直接使用session framework。

下一步驟，
session跟authentication系統。大部分時間，你不會需要所有的功能，但是當你需要在使用者之前進行複雜的處理的時候，他對你是很有幫助的。
接下來將會看看django 的cache架構，這是很方便的方法去改進你應用程式的效能。


django.contrib

Python強大的優點裡其中一項就是 "Batteries included"哲學：當你安裝好python，他就具備大量你可以使用的標準函式庫，不需要額外下載。Django也遵循這樣的哲學思維，因此他包含了許多自己額外有用的標準函式庫在網頁開發工作上面。這一章節將會涵括這些部分。

Django 標準函式庫

Django的標準函式庫存在 django.contrib。在他下面的子項目都是各自獨立的函式。這些函式彼此之間不一定有關聯，但是一些 djaongo.contrib 子項目可能彼此之間是有使用到的。

django.contrib的packages中為一個共同點是：如果打算移除整個django.contrib packages，你依然可以使用Django的基本功能。當Django開發人員新增新的功能到framework當中，他們根據這個法則決定是否新的功能需要放在 django.contrib或者是其他地方。

django.contrib 包含這些packages：

admin：Django的admin網站。
admindocs：自動產生的Django admin網站的文件，本書當中不會提到，請自行參閱Django document。
auth：Django的認證系統。
comments：回應的應用程式。
contenttypes：一個可以對內容的型別進行處理的框架，每一個安裝的django model都是一個各自獨立的conten type。這個框架通常是被其他 contrib 應用程式拿來使用，而且幾乎都是進階的Django 開發人員。這些開發人員應該可以透過閱讀這個應用程式的原始碼(django/contrib/contenttypes)找到更多資訊。
csrf：保護避免跨網域攻擊(CSRF)。參閱後面的CSRF保護章節。
databrowse：一個讓你瀏覽資訊的Django的應用程式。這本書不會包含這個功能。
flatpages：一個可以透過資料庫來管理HTML頁面的框架。參閱後面的章節 Flatpages。
formtools：一個具備多個有用的處理表格的高階函式。
gis：讓Django增加GIS(Geographic Information Systems)支援的功能。例如，允許你的Django Model可以儲存地理資訊並且進行查詢。這是一個龐大且複雜的函式庫。可以從http://geodjango.org 獲得更多資訊。
humanize：一組Django樣板過濾器，可以讓資訊變得比較容易閱讀。參照後面的章節 ”Humanizing Data”。
localflavor：針對不同國家或者是文化進行數字處理。通常用來驗證美國的區碼或者是身分字號。
markup：一組Django的樣板過濾器，實作一組通用的標示語言。參閱後面的區塊 “標記過濾器”
redirects：用來處理redirects的框架。參閱後面的區塊 “Redirects”
sessions：Django的session框架。
sitemaps：產生sitemap的框架。
sites：讓你很多個網站對同一個資料庫進行操作的框架。參閱後面的部份 “Sites”
syndication：處理RSS跟Atom的框架。
webdesign：Django額外提供的，對於網頁設計人員相當有幫助。目前只有一個template標籤，{% lorem %}。

其他的章節將會持續探討這一章節所沒包含的 django.contrib的package。

Sites

Django的sites系統是一個通用的框架，讓你可以運作多個網站，但是對同一個資料庫跟Django專案做處理。這是一個很抽象的說法，但是可以有技巧的方式去理解他。透過幾個情境來了解他是多麼的有用。

情境一：在多個網站重複使用資訊
在第一章裡面提到，利用Django開發的網站 LJworld.com 跟 Lawrence.com 是同一個組織所在運作：Lawrence Journal-world newpapaer在Lawrence, Kansas。LJWorld.com專注新聞，Lawrence.com專注在地方娛樂。但是有時候編輯想要再兩個網站發佈同一篇文章。
沒有經過大腦的方法來解決這個問題就是使用分開的兩個資料庫，並且要求網站人員發佈同一篇文章兩次：一篇給LJWorld.com，然後再Lawrence.com在發佈一篇。但是這對於網站人員來說是沒有效率的，而且在資料庫當中會重複儲存一樣的資訊。

比較好的方法呢？兩個網站使用同一個文章的資料庫，而且一篇文章跟一個或多個網站利用many-to-many的關係產生關聯。Django sites框架可以對資料庫表格中的文章做關聯。這就好像對資料跟一個或多個網站進行關聯。

情境二：儲存你的網站名稱跟網域在同一個地方
LJWorld.com 跟 Lawrence.com 都有e-mail警示通知的功能，讓讀者可以註冊並且在有新聞發生的時候取得通知。非常的基本：一個讀者填寫表單，然後他馬上就會收到一封信件說，”謝謝你的訂閱”。

重複實作註冊流程的程式碼是沒有效率的，比較合理的作法應該是網站使用同一個程式碼。但是 “謝謝你的訂閱”的訊息，要注意兩個網站是不同的。利用Site物件，我們可以將感謝的訊息抽象化，利用目前網站的名稱跟網域的資訊取代。

Django網站框架提供一個地方讓你儲存每個網站的 name 跟 domain 在你的Django的專案當中，這意味著你可以重複使用這些資訊在一般情況。

如何使用 Sites 框架
sites 框架其實多個習慣設定，還稱不上framework。
這整件事基於兩個基本的慨念：
Site的model可以在 django.contrib.sites 中找到，包含 domain 跟 name的欄位。
SITE_ID 的設定可以指定存在資料庫裡面的ID的Site物件跟部分的設定檔產生關聯。

要怎麼使用這兩個概念都可以，但是Django透過簡單的習慣設定使用他們。
依照下面的步驟來安裝sites應用：
新增 ‘django.contrib.sites’ 到你的 INSTALLED_APPS。
執行 manage.py syncdb，會自動安裝 django_site 表格到你的資料庫當中。這也會建立預設的 site 物件，預設網域為 ‘example.com’。
改變 ‘example.com’ 網站為你自己的網域，透過 Django admin或者是 Python的API新增其他的 Site 物件。建立一個 Site 物件給每個 網站/網域
定義 SITE_ID 設定在你的每個設定檔案。這個ID數值用來決定哪個網站對應哪個存在資料庫中的 Site 物件。

Sites 框架的相容性
這部份告訴你還可以利用sites框架做哪些事情。

在多個網站重複使用資訊
剛剛情境一提到的狀況，在多個網站重複使用相同的資訊，只要在你的Model當中對Site
建立一個 ManyToManyField的欄位即可，例如：
from django.db import models
from django.contrib.sites.models import Site

class Article(models.Model):
    headline = models.CharField(max_length=200)
    # ...
    sites = models.ManyToManyField(Site)

當你需要在不同網站之間對你的文章進行關聯的時候，架構上應該是這樣。你可以重複使用相同的Django view程式碼給不同的網站。繼續剛剛的 Article model的範例，這邊有一個叫做 article_detail的 view 應該是長這樣：

from django.conf import settings
from django.shortcuts import get_object_or_404
from mysite.articles.models import Article

def article_detail(request, article_id):
    a = get_object_or_404(Article, id=article_id, sites__id=settings.SITE_ID)
    # ...


這個View函式可以重複使用，因為他根據SITE_ID的設定數值來動態去檢查該文章的網站。
舉例來說，LJWorld.com的設定檔有一個SITE_ID設為1，而Lawrence.com的設定檔的SITE_ID設為2。如果這個view被呼叫到，當LJWorld.com的設定檔案是啟動的，他將會限制文章在查找的時候是針對LJWorld.com的文章。

對單一網站做內容的關聯
類似的情況，你可以利用 ForeignKey 讓一個 model 對 Site 建立 many-to-one 的關聯。
舉例來說，如果每篇文章只跟一個網站做關聯，你使用的model應該是長這樣：
from django.db import models
from django.contrib.sites.models import Site

class Article(models.Model):
    headline = models.CharField(max_length=200)
    # ...
    site = models.ForeignKey(Site)
這個效果跟我們最後提的一樣。

從Views快勾入目前網站
在更低階的情況，你可以使用sites框架在你的Django的View當中做一些事情，根據你目前被呼叫的網站，例如：
from django.conf import settings

def my_view(request):
    if settings.SITE_ID == 3:
        # Do something.
    else:
        # Do something else.

當然，像這樣直接將網站ID寫死是很醜的。一個比較高明的作法達到同樣的目的是去檢查目前網站的網域：
from django.conf import settings
from django.contrib.sites.models import Site

def my_view(request):
    current_site = Site.objects.get(id=settings.SITE_ID)
    if current_site.domain == 'foo.com':
        # Do something
    else:
        # Do something else.

利用 settings.SITE_ID 取得Site物件是相當普遍的，所以 Site 的 manager (Site.objects)具備一個get_current()的方法。這個例子的結果會跟上一個相同：
from django.contrib.sites.models import Site

def my_view(request):
    current_site = Site.objects.get_current()
    if current_site.domain == 'foo.com':
        # Do something
    else:
        # Do something else.


注意：在最後這個例子當中，你不需要 import django.conf.settings

為了顯示來取得目前網域

基於DRY (Don’t Repeat Yourself) 的方法來取得你網站的名稱跟網址，如先前解釋過的 “情境二：儲存你的網站名稱跟網域在一個地方”，只要利用 Site 物件就可以參考到名稱跟網域了，舉例來說：
from django.contrib.sites.models import Site
from django.core.mail import send_mail

def register_for_newsletter(request):
    # Check form values, etc., and subscribe the user.
    # ...
    current_site = Site.objects.get_current()
    send_mail('Thanks for subscribing to %s alerts' % current_site.name,
        'Thanks for your subscription. We appreciate it.\n\n-The %s team.' % current_site.name,
        'editor@%s' % current_site.domain,
        [user_email])
    # ...
繼續從 LJWorld.com跟Lawrence.com的例子中往下看，在 Lawrence.com 這封信件中的標題是 “謝謝訂閱 lawrence.com 的通知信”。在 LJWorld.com，通知信件的標題會是 “謝謝訂閱 LJWorld.com 的通知信”。相同的專屬於網站的行為都被應用在郵件的內容當中。

還可以有更彈性(比較複雜一點)的作法來達成這件事情就是使用 Django 的樣板系統。
Lawrence.com 跟 LJWorld.com 有不同的樣板資料夾( TEMPLATE_DIRS)，你可以像這樣簡單的配置給樣板系統：
from django.core.mail import send_mail
from django.template import loader, Context

def register_for_newsletter(request):
    # Check form values, etc., and subscribe the user.
    # ...
    subject = loader.get_template('alerts/subject.txt').render(Context({}))
    message = loader.get_template('alerts/message.txt').render(Context({}))
    send_mail(subject, message, 'do-not-reply@example.com', [user_email])
    # ...
在這個情況下，你必須要建立 subject.txt 跟 message.txt 樣板在 LJWorld.com 跟 Lawrence.com 兩個網站的樣板資料夾當中。如同剛剛所說的，這是更有彈性的作法，但是也有點複雜。
盡可能的利用Site物件來免除不必要的複雜度跟重複性，是不錯的主意。

CurrentSiteManager
如果Site物件在你的應用程式當中扮演重要的角色，那麼試著在Model中使用 CurrentSiteManager。這是一個 model manager，自動過濾查詢當中的結果找出符合目前網站的物件。
藉由在你的Model當中例外增加CurrentSiteManager來使用他。例如：
from django.db import models
from django.contrib.sites.models import Site
from django.contrib.sites.managers import CurrentSiteManager

class Photo(models.Model):
    photo = models.FileField(upload_to='/home/photos')
    photographer_name = models.CharField(max_length=100)
    pub_date = models.DateField()
    site = models.ForeignKey(Site)
    objects = models.Manager()
    on_site = CurrentSiteManager()

在這個model當中， Photo.objects.all() 將會回傳所有的在資料庫當中的 Photo的物件，但是Photo.on_site.all() 將會根據 SITE_ID的設定值回傳跟目前網站相關連的 Photo物件。

換句話說，下面兩個描述的內容結果是一樣的：
Photo.objects.filter(site=settings.SITE_ID)
Photo.on_site.all()

CurrentSiteManager 是如何知道Photo裡面哪個欄位是Site呢？預設會去找尋 site 的欄位。如果你的model當中具備一個 ForeignKey 或者是 ManyToManyField 但是不叫做 site ，你需要特別傳遞他當做參數給 CurrentSiteManager 。接下來的Model會是一個具有 publish_on的欄位，長的像這樣：
from django.db import models
from django.contrib.sites.models import Site
from django.contrib.sites.managers import CurrentSiteManager

class Photo(models.Model):
    photo = models.FileField(upload_to='/home/photos')
    photographer_name = models.CharField(max_length=100)
    pub_date = models.DateField()
    publish_on = models.ForeignKey(Site)
    objects = models.Manager()
    on_site = CurrentSiteManager('publish_on')

如果你常試著去使用 CurrentSiteManger 然後傳遞一個不存在的欄位名稱，Django將會觸發一個ValueError的錯誤。

注意：
你可能想要在你的Model當中具備一個普通的Manager(不是Site的)，即使你使用CurrentSiteManger。在附錄B當中解釋提到，如果你自己定義一個Manager，Django將不會自動建立 objects = models.Manager() 的 manager給你。

而且，Django的某些部分裡，特別是，Django的Admin site跟一般的views當中，會優先使用最先在model當中被定義的manager，所以如果你想要你的admin site去存取所有的物件，(而不是只有特定網站的)，將 objects = models.Manager() 放在你的model當中，而且在你定義 CurrentSiteManager之前。

Django如何使用Sites框架
雖然你不一定會需要使用sites框架，但是強烈推薦，因為Django在某些情況可以獲得一些好處。即使如果你安裝Django的目的只是為了單一網站的設計，你應該也要花一點時間用domain跟name建立你的site物件，並且將他的ID設定到SITE_ID設定裡。

這邊是Django如何使用site 框架：
在重新導向的框架(redirects framework)中(參閱後面的Redirects章節)，每次的頁面重導都會被某個site中有關聯。當Django在尋找一個重導的連結的時候，他會去找尋目前的SITE_ID。
在回應框架(comment framework)中，每次的回應都會跟某個site有關聯。當一個回應被張貼出來，他的site資訊就會被記錄為SITE_ID，而且當回應被某些特定的樣板標籤列出的時候，只有屬於該site的回應會被列出來。
在(flatpages framework)中(參閱後面的Flatpages章節)，每個flatpages都會被某個site有關聯。當一個flatpages被建立的時候，你會需要指定他的site，而且flatpages middleware為了取得flatpages的資訊就會去檢查目前SITE_ID。
在訂閱框架(Syndication framework)中(參閱第十三章)，樣板中的title跟description自動就具備存取一個{{ site }}變數，也就是Site這個物件來呈現目前site用。而且，如果你不完整指定特定的網址資訊，他會從目前的Site物件當中取得domain的資訊當做文章項目的URL。
在認證系統(authentication framework)中(參閱第十四章)，django.contrib.auth.views.login 的view會將目前的Site名稱以 {{ site_name }} 的樣貌傳遞到樣板當中，而目前的Site物件則是 {{ site }}。

Flatpages
通常你都會需要一個資料庫導向的網頁應用程式來運作，但是你也會需要增加一些依次性的靜態頁面資訊，例如關於頁面或者是隱私權宣告的頁面。當然你可以使用標準的網頁伺服器例如Apache來處理這些純html頁面檔案，但是這會增加你網站的複雜程度，因為你必須要去另外調整Apache，然後還要去設定存取權限給你的團隊人員來編輯這些檔案，而且你還沒辦法使利用Django的樣板系統來處理頁面。

針對這樣類似問題的解決方法就是Django的flatpages應用程式，他就位於 django.contrib.flatpages。這個應用程式讓你可以直接透過Django的管理後台來管理類似的一次性的頁面。而且他讓你可以使用Django的樣板系統指定特定的樣板。他背後是利用Django的model，這代表說他將這些頁面存在資料庫當中，就如同你其他的資訊一樣，而且你可以透過database api來存取flatpages。

Flatpages是以site跟URL為依據。當你建立一個flatpages的時候，你指定一個特定的URL跟site。(更多關於Site的資訊請參閱Sites章節)

使用Flatpages
依照下面的步驟來安裝flatpages應用程式：
在你的INSTALL_APPS裡面新增 django.contrib.flatpages 。django.contrib.flatpages 需要 django.contrib.sites才能運作，因此確定這兩個套件都有存在 INSTALL_APPS。
新增 django.contrib.flatpages.middleware.FlatpageFallbackMiddleware 到你的 MIDDLEWARE_CLASSES設定中。
執行 manage.py syncdb 的指令來安裝這個必備的表格到你的資料庫當中。

flatpages的應用程式會新增兩個表格在你的資料庫當中：django_flatpage跟django_flatpage_sites。
django_flatpages簡單的將頁面的標題跟文字的內容對應到相關的網址。django_flatpage_sites是一個多對多的表格，將flatpages跟一個或多個site建立關係。

這個應用程式還包含一個FlatPage的model，他被定義在 django/contrib/flatpages/models.py。看起來像是這樣：
from django.db import models
from django.contrib.sites.models import Site

class FlatPage(models.Model):
    url = models.CharField(max_length=100, db_index=True)
    title = models.CharField(max_length=200)
    content = models.TextField(blank=True)
    enable_comments = models.BooleanField()
    template_name = models.CharField(max_length=70, blank=True)
    registration_required = models.BooleanField()
    sites = models.ManyToManyField(Site)

稍微解釋一下這些欄位的意義：
url：這個flatpage在網路上面存在的網址，不包含網域名稱，但是包含前方的斜線(例如：/about/contact/)。
title：flatpage的標題。這個框架不會特地處理這個欄位的資訊。所以你需要自己從頁面中去處理。
content：flatpage的內容(例如：頁面的HTML)。這個框架也不會特別去處理這個欄位裡面的資訊，你一樣必需自己從樣板中去呈現。
enable_comments：是否開啟這個flatpage的回應功能。這個框架不會特別幫你去處理。如果需要你可以自己在樣板中透過檢查這個數值來選擇是否呈現回應。
template_name：這個flatpage在呈現頁面的時候需要的template的名稱。這是選擇性的。如果沒有設定或者是這個樣板不存在，系統會自動使用預設的樣板 flatpages/default.html。
registration_required：是否需要註冊才可以看到這個頁面。這部份跟Django的認證系統/使用者框架做整合，之後會在14章討論到。
sites：這個flatpage存在那一個的site。這部份與之前提到的Django的Site框架做整合。

你可以透過Django的管理後台來建立flatpages或者是透過Django的資料庫api來做。更多資訊可以參閱後面的『新增，變更與刪除Flatpages』。

只要你一件例
