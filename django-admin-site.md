Django Admin site
========
啟動 Djagno 管理後台有五個步驟：
增加 django.contrib.admin 到你的 INSTALLED_APPS 設定當中
選擇你的應用程式當中哪幾個 model 要被管理者介面存取
對這些 models 你可以選擇性的建立 ModelAdmin的類別來封裝客製化的管理功能提供給部分 models的選項。
將 AdminSite實體化並且告訴他你的相關 Models 跟 ModelAdmin 類別
將 AdminSite 的網址寫道你的 URLconf 當中

P.S. Admin 下如果發現有些表格，因為你增加或者是修改 Admin的功能而發生消失的情況，嘗試著將伺服器終止在啟動。

ModelAdmin 物件

class ModelAdmin

ModelAdmin 類別是用來呈現一個 model在管理介面當中的樣子。他們都被存放在你的應用程式裡面的 amdin.py 的檔案當中。簡單的 ModelAdmin 是看起來像這樣：

from django.contrib import admin
from myproject.myapp.models import Author

class AuthorAdmin(admin.ModelAdmin):
    pass
admin.site.register(Author, AuthorAdmin)

你需要 ModelAdmin 嗎？
在前面的例子當中， ModelAdmin 類別並沒有定義任何的數值。結果就是會使用預設的 管理介面的樣子。如果你對於預設的管理介面滿意，你不需要另外去定義 ModelAdmin 物件，你可以註冊你的 model 類別而不需要提供一個 ModelAdmin 的描述。所以先前範例這樣寫會比較精簡一點：
from django.contrib import admin
from myproject.myapp.models import Author

admin.site.register(Author)
 
ModelAdmin 的選項
ModelAdmin很有彈性。他已經具備多個選項來處理客製化的介面。所有的選項都可以被定義在 ModelAdmin 的子類別當中：



class AuthorAdmin(admin.ModelAdmin):
    date_hierarchy = 'pub_date'



ModelAdmin.date_hierarchy
設定 date_hierarchy 到你 model 當中是 DateField 或者是 DateTiemField 欄位，在他們列表時，會攺變成包含一個以日期為主下拉式樣的導覽列。

ModelAdmin.form
預設的  ModelForm 是動態產生給你的 Model。這是用來產生一個表格來處理新增跟變更的頁面。你可以簡單的提供你自己的 ModelForm 來變更預設的表單行為。
相關的範例參閱 新增客製化驗證功能到你的管理介面 。

ModelAdmin.fieldsets
設定 fieldsets 來控制管理介面的新增跟變更的版面。
fieldsets 是一個包含兩個tuples的串列，每一組用來呈現一個在管理介面表單裡面的 <fieldset> 區塊。(一個 <fieldset> 是一個表單的區塊)

這組 tuples 的格式為 (name, field_options) ，name 是呈現該 fieldset 的標題的字串，而 field_options 是一個 dictionary 的格式，包含了該 fieldset 的資訊，所要呈現的欄位。
一個完整的範例可以看一下 django.contrib.flatpages.FlatPage ：

class FlatPageAdmin(admin.ModelAdmin):
    fieldsets = (
        (None, {
            'fields': ('url', 'title', 'content', 'sites')
        }),
        ('Advanced options', {
            'classes': ('collapse',),
            'fields': ('enable_comments', 'registration_required', 'template_name')
        }),
    )


結果就是

￼


如果 fieldsets 沒有提供，Django將會預設顯示非 AutoField 跟 具有 edtiable=True 的欄位，在一個 fieldsets 當中，使用跟在 model 定義當中的順序。

field_options 的 dictionary 可以具備下列幾個 關鍵字：
fields
  一組 tuple 存放要呈現在 fieldset 當中的欄位名稱，這個 key 是必須的。
	例如：
	{
'fields': ('first_name', 'last_name', 'address', 'city', 'state'),
}
如果要顯示多個欄位名稱在一行裡面，將這些欄位包在他們自己的一組 tuple 當中。在這個例子當中 first_name 跟 last_name 將會顯示在同一行裡面。
{
'fields': (('first_name', 'last_name'), 'address', 'city', 'state'),
}
classes
	一個包含額外的 css classes 的串列
	例如：
	{
'classes': ['wide', 'extrapretty'],
}
	兩個有用的 classes 定義給預設的管理介面的樣式表是 collapse 跟 wide。 Fieldset 如果具備了 collapse 樣式在管理介面時，將會被預設為收起來的狀態，並且會用一個小小的連結 “click to expand”。 Fieldsets具備 wide 樣式將會被賦與額外的水平空間。

description
	在每個 fieldset 的上方，fieldset 的下方可以顯示額外的文字說明。
注意，這個欄位的資訊在管理介面當中不會被處理HTML字元跳脫。如果你需要包含HTML的話可以在這邊使用。相對的，你可以使用 django.utils.html.escape() 來跳脫 HTML字元。

ModelAdmin.fields
利用這個選項取代 fieldsets ，如果版面不是很重要，而且如果你想要只顯示表格的欄位就好。舉例來說，你可以用下面的方法來定義一個簡單版本的管理介面表格給 django.contrib.flatpages.FlatPage：

class FlatPageAdmin(admin.ModelAdmin):
    fields = ('url', 'title', 'content')

在上面的例子當中，只會顯示 url, title, 跟 content 。

注意：
不要將 fields 選項跟 fieldsets 選項當中的 fields 關鍵字搞混。

ModelAdmin.exclude
這個屬性，如果被提供，應該是一組被排除在外的表單欄位名稱。
舉例來說，
class Author(models.Model):
    name = models.CharField(max_length=100)
    title = models.CharField(max_length=3)
    birth_date = models.DateField(blank=True, null=True)

如果你想要提供一個 Author 的表格只有 name 跟 title 欄位，你應該利用下面的方法來指定 fields 或者是 exclude ：
class AuthorAdmin(admin.ModelAdmin):
    fields = ('name', 'title')

class AuthorAdmin(admin.ModelAdmin):
    exclude = ('birth_date',)

當 Author 只有具備三個欄位, name, title, birth_date 這個表格利用上面的宣告方式結果將會一樣。

ModelAdmin.filter_horizontal
使用一個小巧且利用非侵入式 Javascript 的過濾器介面來取代舊有的 <select mutiple>功能。這個數值是一個包含要作用的欄位的串列。將會以水平的方式呈現。可以看看 filter_vertical 來使用垂直式的介面。

ModelAdmin.fitler_vertical
跟 filter_horizontal 一樣，但是是垂直的過濾器介面。

ModelAdmin.list_display
設定 list_display 來控制哪些欄位要被顯示在變更的管理介面列表。

list_display = ('first_name', 'last_name')

如果你沒有設定 list_display，管理介面將會顯示一個單一的欄位，並且以 __unicode__() 的結果來呈現該物件。
你可以有四個可能的參數在 list_display 中使用：
model 中的欄位
	class PersonAdmin(admin.ModelAdmin):
    		list_display = ('first_name', 'last_name')

一個可以呼叫，給 model 實體接受一個參數
	def upper_case_name(obj):
    		return ("%s %s" % (obj.first_name, obj.last_name)).upper()
		upper_case_name.short_description = 'Name'

	class PersonAdmin(admin.ModelAdmin):
    		list_display = (upper_case_name,)

用一個字串代表一個 ModelAdmin 的屬性。就如同一個可呼叫的函式。
	class PersonAdmin(admin.ModelAdmin):
    		list_display = ('upper_case_name',)

    		def upper_case_name(self, obj):
      		return ("%s %s" % (obj.first_name, obj.last_name)).upper()
    		upper_case_name.short_description = 'Name'

一個字串來代表 Model 的某個屬性。類似一個可被呼叫的，但是他的 self 是指他自己 model 本身的實體。
	class Person(models.Model):
		name = models.CharField(max_length=50)
    		birthday = models.DateField()

    		def decade_born_in(self):
        		return self.birthday.strftime('%Y')[:3] + "0's"
    			decade_born_in.short_description = 'Birth decade'

	class PersonAdmin(admin.ModelAdmin):
    		list_display = ('name', 'decade_born_in')


關於 list_display 有幾個地方要注意的：
如果該欄位是一個 ForeignKey，Django 將會顯示他關連的物件的 __unicode__() 資訊。
不支援 ManyToManyField 欄位，因為那將會連貫的執行表單中每一筆資料的 SQL 查詢的動作。如果無論如何都要這樣做，給你的 model 自訂一個方法，然後增加那個方法到 list_display當中。（也就是前面提到的四個方法最後一個。）
如果該欄位是一個 BooleanField 或者是 NullBooleanField ，Django將會顯示一個漂亮的 “開啟”，”關閉”的圖示來取代 True 或者是 False。
如果提供的字串是一個 Model 的方法，ModelAdmin 或者是一個可以被呼叫的物件，Django 預設將會自動將 HTML 字元跳脫。如果你不想要讓該方法被跳脫，將該方法中增加一個屬性為 allow_tags ，並且設定為 True。
class Person(models.Model):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)
    color_code = models.CharField(max_length=6)

    def colored_name(self):
        return '<span style="color: #%s;">%s %s</span>' % (self.color_code, self.first_name, self.last_name)
    colored_name.allow_tags = True

class PersonAdmin(admin.ModelAdmin):
    list_display = ('first_name', 'last_name', 'colored_name')

如果提供的字串是一個 Model 的方法，ModelAdmin 或者是一個可以被呼叫的物件，他會回傳 True 或者是 False， 如果你在該方法當中設定一個 boolean 的屬性，並且將他的術直設定為 True，Django將會顯示一個漂亮的 “開啟” 或者是 ”關閉”的圖示。
	class Person(models.Model):
    first_name = models.CharField(max_length=50)
    birthday = models.DateField()

    def born_in_fifties(self):
        return self.birthday.strftime('%Y')[:3] == '195'
    born_in_fifties.boolean = True

class PersonAdmin(admin.ModelAdmin):
    list_display = ('name', 'born_in_fifties')

__str__() 跟 __unicode__() 是被允許的，就跟其他的 model 方法一樣，所以下面這樣的作法是沒問題的：
	list_display = ('__unicode__', 'some_other_field')

通常，list_display 的元素不是真正資料庫當中的欄位，也不可以被排序。（因為 Django 將所有的排序動作在資料庫的階段處理了。）
然而，如果一個 list_display 中的元素呈現的是某個真正的資料庫欄位，你可以藉由在方法當中將 admin_order_field 屬性設定為該欄位。
例如：
class Person(models.Model):
    first_name = models.CharField(max_length=50)
    color_code = models.CharField(max_length=6)

    def colored_first_name(self):
        return '<span style="color: #%s;">%s</span>' % (self.color_code, self.first_name)
    colored_first_name.allow_tags = True
    colored_first_name.admin_order_field = 'first_name'

class PersonAdmin(admin.ModelAdmin):
    list_display = ('first_name', 'colored_first_name')

上面的例子將會告訴 Django ，當試著去對 colored_first_name 進行排序的時候，他會根據 first_name 來做排序。




