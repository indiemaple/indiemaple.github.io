## 在 Laravel 5.3 中使用官方扩展包 Scout + Algolia 实现全文搜索实例教程

今天，我来给大家演示下如何在Laravel 5.3中使用 Scout + Algolia 实现全文搜索。

Laravel 5.3 为我们提供了很多新特性，同时也引入了很多新的扩展包，今天我们就要用到 Laravel 5.3 提供的基于模型实现全文搜索的 Scout 扩展包。如果你想要在 Laravel 中实现全文搜索功能，那么最快捷的办法也就是使用这个扩展包。

在本教程中，我将手把手一步一步进行操作和说明，以便大家更好的理解其原理和实现，从而在自己的应用中更加得心应手地去使用。

## 首先，给大家放一个预览图：
![image](/assets/image/laravel-scout.jpg)

接下来，我们一步步庖丁解牛般给大家复盘其实现。
##### 第一步：安装 Laravel 5.3 应用
如果还没有安装 Laravel 5.3 的话首先肯定是要安装它，毕竟巧妇难为无米之炊，得益于 Composer，安装过程也是极其简单，只需要在 web 根目录下执行以下命令即可：

    composer create-project --prefer-dist laravel/laravel blog

##### 第二步：安装扩展包
工欲善其事，必先利其器。在这一步中，我们需要安装两个扩展包：  
1）laravel/scout  
2）algolia/algoliasearch-client-php  
首先，我们通过 Composer 来安装 Scout，在项目根目录下运行如下安装命令：

    composer require laravel/scout

安装成功之后，打开config/app.php注册服务提供者：
```php
'providers' => [
    ....
    Laravel\Scout\ScoutServiceProvider::class,
]
```

接下来，我们还要通过以下命令发布配置文件：  

    php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"

运行完成后，会发现在config目录下新增了一个scout.php文件，该文件就是 Scout 的配置文件。

好了，接下来我们来安装algolia扩展包：

    composer require algolia/algoliasearch-client-php

##### 第三步：扩展包配置
安装完 Algolia 扩展包之后，还需要配置 Algol 的 id 和 secret。这需要我们在Algolia.com上有一个账户才能获取到对应的信息，如果还没有的话去注册一个。

注册完成登录后通过这个链接获取到应用的 id 和 secret 信息，web界面和信息所处位置如下：
![image](/assets/image/algolia.jpg)
将上述 id 和 secret 信息设置到.env配置文件：

    ALGOLIA_APP_ID=paste app id
    ALGOLIA_SECRET=paste app secret

##### 第四步：创建数据表和模型
做好以上准备工作之后，我们需要通过 Laravel 提供 Artisan 命令为 items 表创建迁移：  

    php artisan make:migration create_items_table

运行完此命令之后，会在database/migrations目录下看到新生成的迁移文件，编辑该文件内容如下：

```php
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateItemsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('items', function (Blueprint $table) {
           $table->increments('id');
           $table->string('title');
           $table->timestamps();
       });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::drop("items");
    }
}
```

创建完items表之后，还要为该表创建对应的模型类。我们在app目录下创建Item.php文件并编辑其内容如下：
```php
<?php
namespace App;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class Item extends Model
{

    use Searchable;

    public $fillable = ['title'];

    /**
     * 获取模型的索引名称
     *
     * @return string
     */
    public function searchableAs()
    {
        return 'items_index';
    }
}
```
##### 第五步：添加新的路由
在这一步中我们需要为新增items及其列表添加路由。我们在routes/web.php文件中添加如下路由：
```php
Route::get('items-lists', ['as'=>'items-lists','uses'=>'ItemSearchController@index']);
Route::post('create-item', ['as'=>'create-item','uses'=>'ItemSearchController@create']);
```

##### 第六步：创建控制器
完成上述操作后，接下来我们来创建路由中定义的控制器app/Http/Controllers/ItemSearchController.php，该控制器用于管理items列表以及新增item并返回响应。因此，我们编辑该控制器文件内容如下：
```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Http\Requests;
use App\Item;

class ItemSearchController extends Controller
{

    /**
     * items列表
     */
    public function index(Request $request)
    {
        if($request->has('titlesearch')){
            $items = Item::search($request->titlesearch)
                     ->paginate(6);
        }else{
            $items = Item::paginate(6);
        }
        return view('item-search',compact('items'));
    }


    /**
     * 创建新的item
     */
    public function create(Request $request)
    {
        $this->validate($request,['title'=>'required']);

        $items = Item::create($request->all());
        return back();
    }
}
```
##### 第七步：创建视图
最后一步，我们来创建视图文件resources/views/item-search.blade.php，编辑视图文件代码如下：
```html
<!DOCTYPE html>
<html>
    <head>
        <title>Laravel 5.3 - laravel scout algolia search example</title>
        <link rel="stylesheet" type="text/css" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css">
    </head>
    <body>
        <div class="container">
            <h2>Laravel Full Text Search using Scout and algolia</h2><br/>
            <form method="POST" action="{{ route('create-item') }}" autocomplete="off">
            @if(count($errors))
            <div class="alert alert-danger">
                <strong>Whoops!</strong> There were some problems with your input.
                <br/>
                <ul>
                @foreach($errors->all() as $error)
                    <li>{{ $error }}</li>
                @endforeach
                </ul>
            </div>
            @endif

            <input type="hidden" name="_token" value="{{ csrf_token() }}">

            <div class="row">
                <div class="col-md-6">
                    <div class="form-group {{ $errors->has('title') ? 'has-error' : '' }}">
                        <input type="text" id="title" name="title" class="form-control" placeholder="Enter Title" value="{{ old('title') }}">
                        <span class="text-danger">{{ $errors->first('title') }}</span>
                    </div>
                </div>
                <div class="col-md-6">
                    <div class="form-group">
                        <button class="btn btn-success">Create New Item</button>
                    </div>
                </div>
            </div>
        </form>

        <div class="panel panel-primary">
            <div class="panel-heading">Item management</div>
                <div class="panel-body">
                    <form method="GET" action="{{ route('items-lists') }}">
                    <div class="row">
                        <div class="col-md-6">
                            <div class="form-group">
                                <input type="text" name="titlesearch" class="form-control" placeholder="Enter Title For Search" value="{{ old('titlesearch') }}">
                            </div>
                        </div>
                        <div class="col-md-6">
                            <div class="form-group">
                                <button class="btn btn-success">Search</button>
                            </div>
                        </div>
                    </div>
                    </form>

                    <table class="table table-bordered">
                        <thead>
                             <th>Id</th>
                             <th>Title</th>
                             <th>Creation Date</th>
                             <th>Updated Date</th>
                        </thead>
                        <tbody>
                        @if($items->count())
                            @foreach($items as $key => $item)
                            <tr>
                                <td>{{ ++$key }}</td>
                                <td>{{ $item->title }}</td>
                                <td>{{ $item->created_at }}</td>
                                <td>{{ $item->updated_at }}</td>
                            </tr>
                            @endforeach
                        @else
                            <tr>
                                 <td colspan="4">There are no data.</td>
                            </tr>
                        @endif
                        </tbody>
                    </table>
                    {{ $items->links() }}
                </div>
            </div>
        </div>
    </body>
</html>
```
OK，现在我们可以来看看上述代码的效果了，首先在项目根目录下启动PHP内置服务器：

    php artisan serve

然后在浏览器中访问 items 列表页：

    http://localhost:8000/items-lists

如果你已经添加记录到items数据表的话，就可以运行如下命令将数据添加到索引：

    php artisan scout:import "App\Item"

这样就可以在页面中进行搜索操作了。

想要了解更多 Scout 扩展包的使用细节，可参考对应中文文档：http://laravelacademy.org/post/6277.html

> 声明：本文整理翻译自[itsolutionstuff.com](http://itsolutionstuff.com/)
