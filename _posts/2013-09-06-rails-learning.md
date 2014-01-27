---
layout: post
title: "Rspec 学习记录"
description: "Ruby 学习记录"
category: "Ruby Rails"
tags: [Ruby, Rails, Rspec]

---


##Rspec

1. Rspec 利用 Ruby 定义了一套领域专属语言**DSL**
2. 对 controller 和 model 做rspec 时，首个测试描述需要将各自的 ControllerName
   或者 ModelName 写出来，不能加双引号:

  ```ruby
    describe RelationshipsController do
      ...
    end
  ``````
3. Rails 中的约定，用 `user` 代替 `user.id`，Rails 会自动获取用户的 id
4. 在每个元素上调用同一个方法的情况是很常见的，所以 Ruby 为此定义了一种简写形式，在 & 符号后面跟上被调用方法

  ```ruby
  >> [1, 2, 3, 4].map { |i| i.to_s }
  => ["1", "2", "3", "4"]
  >> [1, 2, 3, 4].map(&:to_s)
  => ["1", "2", "3", "4"]
  ```

  获取 user.followed_users 集合中所有用户的 id 可通过 `User.first.followed_users.map(&:id)` 或者
  `User.first.followed_user_ids`实现.

  其中`followed_user_ids` 是 **Active Record** 根据 has_many :followed_users 关联合成的，
  这样我们只需在关联名的后面加上 `_ids` 就可以获取 `user.followed_users` 集合中所有用户的 id 了
5. **Guard自动测试**可以对spec/ 文件变化进行自动测试, 操作步骤如下:

    a. gem 中加入 'guard-rspec'
    b. $bundle install
    c. $bundle exec guard init rspec
    d. 修改 Guardfile

        require 'active_support/inflector'

        #保证失败的测试通过后不继续执行，以提高测试速度
        guard 'spec', all_after_pass: false do
            ...
        end

    e. $bundle exec guard
    f. 配置结束，开始享受Guard带来的清凉
6. Spork + Guard 测试配置
    a. 导入 Spork 配置文件 `$Spork --bootstrap`
    b. 执行 Spork `$spork`
    c. `$rspec spec/` 或者 `$guard`
    d. Guard 和 Spork 协作
        A. `$guard init spork`
        B. 修改Gemfile 文件中 `guard 'spec', all_after_pass: false do` 为
        `guard 'spec', all_after_pass: false, cli: '--drb' do`

7. Rspec约定如果对象有一个返回`bool` 值的方法`fool`，就有一个对应的`be_fool` 可在测试中使用
8. `expect { click_button "Create my account" }.not_to change(User, :count)`, 其中的`click_button`
  动作会前后计算`User.count` 并比较变化
9. Spec中 `have_selector` 可测试HTML标签是否出现在页面中
10. `let` 相对 `let!` 是隋性的，只有到 `let` 的对象要真正使用的时候才创建对象


-------------------------------------------------

##HTML
1. Embeded Ruby ==> .erb Rails 识别的模板文件后缀名
2. `<a href="*****">Ruby on Rails </a>`， 生成一个链接
3. `<% provide(:title, "Home") %>`， `provide` 调用Rails代码
4. `<%= %>`, 执行代码并插入到模板中去
5. CSS 规则通过 class, id, HTML标签三者结合定义，所有 HTML 元素都可指定 class 和 id. class 在一个网页中可多次使用，因为 id 是唯一的，只能使用一次。
6. `<%= link_to "Home", '#‘ %>`, `link_to`创建链接， “Home” 为链接的文件， '#'为占位符，填入链接地址可链接到指定地址
7. `nl` 无序列表标签
8. `li` 列表项目标签
9. `div` 包含的是块级元素， 两个 `div` 之间是换行的
9. Assert pipeline 框架在`app/asserts/images`中找寻所需图片。 `image("rails.png", alt: "Rails")`, 其中`:alt`属性在图片无法加载时显示图片名
10. bootstrap twitter 框架与 Assert pipeline 兼容使用，在*config/application.rb*中加入`config.assets.precompile += %w(*.png *.jpg *.jpeg *.gif)`， 以防止bootstrap被Assert pipeline covered。
11. *app/asserts/stylesheets* 是Assert pipelinee 的一部分，该目录下所有样式表都会被 Assert pipeline 自动优化后并自动包含在网站的*application.css*中
12. Sass 是生成css的工具， *custom.css.sass*表示Sassy的css文件。
13. CSS中“#”规则是给id添加样式的， @extend可引入外部的样式规则集合
14. `<div class="alert alert_<%= key %>" <%= value %>></div>`等价于
    `<%= content_tag(:div, value, class:"alert alert_#{key}") %>`
15. `<div class="small-8 columns">`columns 表示是纵向布局页面
17. `</p>`换段，`</br>` 换行


------------------------------------------------


##Rails MVC
*. Model（模型）：代表应用系统的数据信息以及操作这些数据的规则；</p>
*. View（视图）：代表应用系统的用户接口，通常是包含嵌入式 Ruby 代码的 HTML 文件，用于向浏览器提供数据的工作；</p>
*. Controller（控制器）：主要负责处理 Web 请求，检索模型数据并按要求将数据传递给视图。控制器是模型和视图的联系桥梁。</p>
###控制器与视图间的数据传递方法
用户简单的从浏览器上创建表单的过程就包括了将数据从表单传递到控制器，进而传递到模型完成存储的过程。</p>
params 是哈希表，它是 Rails 里最重要的传递参数之一，包含浏览器请求传递的数据。正是这个 params 将数据从客户端表单传递到控制器。Rails 应用程序可以用两种方法的把数据传给 params，进而由控制器的某个方法获取。一种是将用户输入到 HTML 表单的数据封装到 HTTP 请求，通过 POST 方法将数据传给控制器的 params。上文采用的就是这种方法，将 new.html.erb 视图的表单数据通过 POST 方法提交。另外一种方法就是通过把数据作为 URL 的一部分，位于 URL 中 ? 之后，通常称为 URL 查询字符串参数。Rails 并没有对这两种方法传递的数据做出区分，控制器都能够以前文述及的方法从 params 中获取数据。</p>

###控制器与模型间的数据传递
Rails 中模型的属性是靠命名规则与数据库字段实现关联的。模型数据的属性定义是在对应的 migrate 文件中。控制器直接使用 create，save 和 update 等继承自父类 ActiveRecord::Base 的方法，实现数据创建、存储和更新，不需要额外的编写代码。</p>在 Rails 中，控制器获取模型数据过程也是相当的方便快捷。打开控制器源文件：

    def update
      @article = Article.find(params[:id])
      ...
    end

  Rails 按照路由规则把请求 URL 中的 ID 值，通过 params 传递给控制器,Article.find 方法根据 :id 查找现有的数据，然后使用 update_attributes 方法更新记录。Rails 控制器正是通过 ActiveRecord 的 find 方法从模型中读取数据。

###视图内的数据共享
####视图与布局间的数据共享
在 Rails 3 之后，通过脚手架生成的视图布局位于 `layouts/application.html.erb`，能够被所有的控制器共享。布局和视图可以共享控制器中的数据。也就是说，布局可以用与视图完全一样的方法获取控制器中的数据。</p>
布局**可以**读取一些视图中的信息，但布局是不能向视图传递信息的，因为视图的执行是在布局引入或者说发生作用之前。</p>

####视图之间的数据
视图之间经常需要传递数据，Rails 为此进行了精心设计。它不仅提供特有的 `flash` 快捷机制，也支持传统的 `Session` 和 `Cookie` 功能。需要说明的是，此节讲述的数据传递过程，__本质上还是从控制器到视图的__。因为对用户而言，数据是从某个视图获取，然后通过另外一个视图显示出来。因为中间的控制器对开发者而言近乎透明，所以姑且将此类型定义为视图之间的数据传递。</p>
Rails 的 `flash` 机制尤其适用于前后关联的视图页面之间传递数据。用于在两个actions间传递临时数据，flash中存放的所有数据会在紧接着的下一个action调用后清除, 一般用于传递提示和错误消息。从本质上看， flash 是 Rails 内建的一个哈希表，用于从一个控制器向另一个控制器的视图发送用户所需的消息。当控制器调用 `redirect_to` 时，重定向会清空除 flash 之外的所有变量。通常，添加到 flash 的数据能够在紧接着的下个请求里读取。但是某些情况下，您想在当前请求中直接使用。比如文章创建失败，直接渲染页面的情况下，就不会导致新的请求。如果此时需要显示 flash 中的信息，可以用 `flash.now` 方法实现。
flash 的使用必须注意几点：

* 首先，为了保证性能， flash 中存储的数据不能过长。
* 其次，flash 的内容只能跨越一次重定向，如果希望数据能够在重定向后依然有效，需要在重定向之前使用`flash.keep`。
* `flash[:alert],flash[:notice]`一般与`redirect_to`一起用，而不能与`render`一起用。
redirect_to是重定向，会重新发起请求，比render多了一次请求。`flash[:alert],flash[:notice]`只会出现在接下面的一个页面中。
而`render`是服务器端转发，客户端不会重新发送请求，比`redirect_to`少了一次请求。所以一旦一起用，结果是接下来两个页面都有flash[:alert],flash[:notice]，第三个页面时才会消失。正确的做法是render搭配`flash.now[:alert]`,`flash.now[:notice]`一起用

Cookie 通常是由服务器端发送给浏览器的，用于存储少量用户信息的文本。基于 Rails 的应用多数情况下不需要直接存取 Cookie，因为 Rails 内置的会话机制或者第三方的插件都能够支持用户信息管理。存储在 Cookie 中的数据可以跨请求甚至跨会话， 所以用户也可以直接使用 Cookie 存取数据。</p>
Rails 应用程序经常使用 session 来实现多个请求之间的数据传递，session 对象在控制器和视图中都可以使用。</p>
从实现的效果上看，flash、cookie 及 session 都能够完成视图间的数据传递。但 flash 一般用于简单的提示消息，而 cookie 及 session 则常见于用户信息管理。

**respond_to** and **respond_with**
`respond_to`指定了资源格式，`respond_with`指定了传输给控件的资源</p>
`respond_with(@user, :location => users_url)`</p></p>
escape_javascript 方法，在 JavaScript 中写入 HTML 代码必须使用这个方法对 HTML 进行转义</p>
`$("#follow_form").html("<%= escape_javascript(render('users/unfollow'))
%>")`注意不要少了前面的 **#**


---------------------------------------


#Routes
##Resources
Resource routing allows you to quickly declare all of the common routes for a given resourceful controller. Instead of declaring separate routes for your **index, show, new, edit, create, update and destroy actions**, a resourceful route declares them in a single line of code.A resource route maps a number of related requests to actions in a single controller.

A single entry in the routing file, such as: `resources: photos` creates 7 different routes in your application, and all mapping to the `PhotoController`

| HTTP Verb | Path |    Action  | Used for
| ----------- ------ ------------
| GET       | /photos           | index   | display a list of all photos
| GET       | /photos/new       | new |return an HTML form for creating a new photo
| POST      | /photos           | create   | create a new photo
| GET       | /photos/:id       | show     | display a specific photo
| GET       | /photos/:id/edit  | edit     | return an HTML form for editing a photo
| PATCH/PUT | /photos/:id       | update   | update a specific photo
| DELETE    | /photos/:id       | destroy   | delete a specific photo

Creating a resourceful route will also expose a number of helpers to the controllers in your application. In the case of resources :photos:

* photos_path returns /photos
* new_photo_path returns /photos/new
* edit_photo_path(:id) returns /photos/:id/edit (for instance, edit_photo_path(10) * returns /photos/10/edit)
* photo_path(:id) returns /photos/:id (for instance, photo_path(10) returns /photos/10)

If you need to use the urls, you should have to set the host parameters.

###Singual resource
Sometimes you want to visit a resource that clients always look up without referencing an ID.`resource :geocoder` be carful of the plural. Except the `index` action, the others the same as before.

###Namespace
`Admin::PostsController` under the directory `app/controllers/admin/` refers to route:

    namespace :admin do
      resources :posts, :comments
    end

If you want to route `/posts` (without the prefix `/admin`) to `Admin::PostsController`, you could use:

    scope module: 'admin' do
      resources :posts, :comments
    end
or, for a single case: `resources :posts, module: 'admin'`.

If you want to route `/admin/posts` to `PostsController` (without the `Admin::` module prefix), you could use:

    scope '/admin' do
      resources :posts, :comments
    end
or, for a single case: `resources :posts, path: '/admin/posts'`.

###Shallow nesting

    resources :posts do
      resources :comments, only: [:index, :new, :create]
    end
    resources :comments, only: [:show, :edit, :update, :destroy]

it's the same to:

    resources :posts do
      resources :comments, shallow: true
    end

There exists two options for scope to customize shallow routes. `:shallow_path` prefixes **member paths** with the specified parameter:

    scope shallow_path: "sekret" do
      resources :posts do
        resources :comments, shallow: true
      end
    end

the following routes:

HTTP Verb |     Path |  Named Helper
--------- | -----   |     --------
GET  |  /posts/:post_id/comments(.:format) |    post_comments
POST |  /posts/:post_id/comments(.:format)  |post_comments
GET  |  /posts/:post_id/comments/new(.:format) |new_post_comment
GET  |  /sekret/comments/:id/edit(.:format) | edit_comment
GET  |  /sekret/comments/:id(.:format)     | comment
PATCH/PUT | /sekret/comments/:id(.:format)  | comment
DELETE  |   /sekret/comments/:id(.:format)  | comment

The `:shallow_prefix` option adds the specified parameter to the **named helpers**:

    scope shallow_prefix: "sekret" do
      resources :posts do
        resources :comments, shallow: true
      end
    end

the following routes:

HTTP Verb |     Path |  Named Helper
--------- | -----   |     --------
GET  |  /posts/:post_id/comments(.:format) |    post_comments
POST |  /posts/:post_id/comments(.:format)  |post_comments
GET  |  /posts/:post_id/comments/new(.:format) |new_post_comment
GET  |  /comments/:id/edit(.:format)    | edit_sekret_comment
GET  |  /comments/:id(.:format)     | sekret_comment
PATCH/PUT | /comments/:id(.:format) | sekret_comment
DELETE  |   /comments/:id(.:format) | sekret_comment

###Creating Paths and URLs From Objects
In addition to using the routing helpers, Rails can also create paths and URLs from an array of parameters. For example, suppose you have this set of routes:

    resources :magazines do
      resources :ads
    end

1. `<%= link_to 'Ad details', magazine_ad_path(@magazine, @ad) %>`
2. `<%= link_to 'Ad details', url_for([@magazine, @ad]) %>`
3. `<%= link_to 'Ad details', [@magazine, @ad] %>`
1/2/3 is the same.

If you wanted to link to just a magazine:

`<%= link_to 'Magazine details', @magazine %>`

For other actions, you just need to insert the action name as the first element of the array:

`<%= link_to 'Edit Ad', [:edit, @magazine, @ad] %>`

###Add more routes
####member
To add a member route, just add a member block into the resource block:

    resources :photos do
      member do
        get 'preview'
      end
    end

This will recognize `/photos/1/preview` with GET, and route to the preview action of `PhotosController`, with the resource id value passed in `params[:id]`. It will also create the `preview_photo_url` and `preview_photo_path` helpers.

Within the block of member routes, each route name specifies the HTTP verb that it will recognize. You can use get, patch, put, post, or delete here. If you don't have multiple member routes, you can also pass `:on` to a route, eliminating the block:

    resources :photos do
      get 'preview', on: :member
    end

You can leave out the `:on` option, this will create the same member route except that the resource id value will be available in `params[:photo_id]` instead of `params[:id]`.

####collection

    resources :photos do
      collection do
        get 'search'
      end
    end

This will enable Rails to recognize paths such as `/photos/search` with GET, and route to the `search` action of `PhotosController`. It will also create the `search_photos_url` and `search_photos_path` route helpers.

Just as with member routes, you can pass `:on` to a route:

    resources :photos do
      get 'search', on: :collection
    end

####new action
To add an alternate new action using the :on shortcut:

    resources :comments do
      get 'preview', on: :new
    end

This will enable Rails to recognize paths such as `/comments/new/preview` with GET, and route to the `preview` action of `CommentsController`. It will also create the `preview_new_comment_url` and `preview_new_comment_path` route helpers.

If you find yourself adding many extra actions to a resourceful route, it's time to stop and ask yourself whether you're disguising the presence of another resource.

####concern
定义`concern`时用`concern`, 用在`do...end`块中时用`concerns`, 直接用在`resources`后面用`concerns:`
    ```
    ZombieOutlaws::Application.routes.draw do
      concern :defensible do |options|
        resources :shotguns, options
        resources :rifles, options
        resources :knives, options
      end

      resources :sheriffs, concerns: :defensible
      resources :gunslingers, concerns: :defensible
      resources :preachers do
        concerns :defensible, except: :destroy
      end
    end
    ```
=====================

##注意点：
1. **Rails 中Gemfile, config, vender目录下更改代码需要重启rails server**
2. 有些数据库可能对存入其中的数据大小写敏感，故在做唯一性判断时存入数据库之前要将数据转换成统一格式

-----------------------

##Rails 操作
1. `rails new demo --skip-bundle` 不安装bundle
2. `rails new demo --skip-test-unit` 不生成`Test::Unit`测试框架对应的Test文件
3. `rails new demo -d=mysql` 使用mysql 作为数据库
3. `rails g scaffold product name price:decimal --skip-stylesheets` 不生成框架
3. `bundle install --without production` 不安装生产环境所需gem
4. `bundle outdated` 查看过期的gem
4. `rails g scaffold User name:string email:string`  **scaffold生成的框架为单数.** 生成User数据模型，生成模型时不用指定`id`由 Rails 自动指定并设为 primary key
5. rake 相当于是给 Ruby 作数据的部署操作，将数据从model中写入到数据库中

        rake db:migrate     迁移
        rake db:rollback    回退
        rake db:migrate VERSION=0 回退到最初版本
        rake db:reset    还原数据库
        RAILS_ENV=production rake assets:precompile
6. `rails g controller StaticPages home help [about] --no-test-framework`
    生成`StaticPages`控件，**控件为复数**， 后面指定的是控件的动作，生成控件的同时更新路由信息;
    `--no-test-framework` 指不生成测试代码(如果前面指定了使用 rspec , 这里就是 rsepc 测试代码)
7. `rails g model User name:string email:string` **模型为单数**，
    Rails生成一个模型时就会自动创建一个db迁移文件，若要改变已存在的模型数据结构，可直接使用`migration`命令创建迁移文件：

    `$rails g migration add_index_to_users_email`

    `$rails g migration add_password_to_users password_digest: string`

7. `rails destroy controller StaticPages` 撤销Rails 生成的控制器`StaticPages`,
  **不会撤销对route.rb文件进行的修改**
8. `raids g integration_test static_pages` 生成集成测试文件
9. Rails 4中的静态文件目录（资源文件）
    a. *app/asserts*: 存放当前应用程序用到的资源文件
    b. *lib/asserts*: 存放开发团队自己开发的代码库用到的资源文件
    c. *vender/asserts*: 第三方代码库用到的资源文件
10. 清单文件
    a. *app/asserts/stylesheets/application.rb*
11. 针对一个文件处理器处理顺序为**从右向左**
12. Rails的`has_secure_password`可提供安全密码的机制, bcrypt-ruby使用耗时因子cost factor设定加密过程的耗时，
    耗时越长越不易破解:*config/environments/test.rb*
13. 用户身份认证机制：
    a. 通过email找到用户： `user = User.find_by(email: email)`
    b. 验证用户密码： `current_user = user.authenticate(password)`验证成功返回一个用户对象，否则返回false
15. Rails 三个环境：生产，测试，开发； Rails server和console默认使用development环境；若要改为production环境：
    ```ruby
    $rails server --environment production
    $rake db:migrate RAILS_ENV= production
    ```
16. `render` 用来插入局部视图，以 *app/views* 为根目录，局部视图以’_‘下划线开头，多个视图中共用的局部视图放在 *app/views/shared/* 目录下面，在控件中使用`render`, 如UsersController控件中 `render "show_follow"` 指的是 *app/views/users/show_follow.html.erb* 文件
17. params 用来浏览器和控件之间的数据，只允许name和email属性传递 `params.require(:user).permit(:name, :email)`
18. 插入模板前，Erb会自动将Symbol转成字符串
19. 登录浏览器就是`create`了一个session, 发送了一个`POST`请求，退出即向`destroy`动作发送了`DELETE`动作.若要限制session中的方法，在路由上做限制。`resources: sessions, only: [:new, :create, :destroy]`
20. `form_for(@user)`, Rails会自动向/users发送`POST`请求
21. 登录表单和注册表单差别是登录表单无session模型
22. 帮助函数默认是可在视图中和控件中使用，被大家所共享
23. HTTP记住用户状态的方法之一是将用户的id保存在记忆权标(remember_token): `session[:remember_token]=user.id`。
    生成记忆权标属性：`rails g migration add_remember_token_to_users`。浏览器中存有记忆权标，在网站页面之间可通过记忆权标获取数据库中的用户记录.
24. 对于创建用户和编辑用户， Rails使用同一套代码构建表单， Rails区分`POST`,`PATCH`请求是依据ActiveRecord中的`new_record?`来检测的。
25. `will_paginate`分页方法需要ActiveRecord::Relation对象支持，`paginate(page: 1)`接受Hash参数，若page不存在，则显示第一页
26. `<%= render @users %>`. Rails会自动将`@users`当作一系列User对象，逐一遍历这些对象，然后使用*_user.html.erb*
27. `admin`方法是ActiveRecord自动生成的，需要添加admin属性：

      `rails g migration add_admin_to_users admin: true`

28. 索引相关：

        #多键索引
        add_index :micro posts, [:user_id, :created_at]
        t.timestamp 自动创建， created_at和updated_at属性无法手工设定，除非是在FactoryGirl中指定

29. `Micropost.where("user_id=?", id)`，其中的?可确保id的值传入底层SQL语句之前做了转义
30. `<%= render partial: "shared/feed_item", collection: @feed_items %>` 等价于

    `<%= render "shared/feed_item", @feed_items %>`
31. Ajax通过唯一的id获取页面中的元素，Ajax可通过向服务器发送异步请求，在不刷新页面的情况下，Ajax就可以更新页面显示的内容。`form_for(…)`更改成`form_for(… remote: true)`就可以使用Ajax
32. Rails 中的save, create, new方法
    1. save：rails中的save其实是create_or_update，新建或修改记录！
    2. new：只是在内存中新建一个对象，操作数据库要调用save方法。
    3. create = new + 执行sql， create中操作一般是先新建一个对象，判断条件合法后存入数据库
    4. build：与new基本相同，多用于一对多情况下。还有一个不同请看使用示例： `Article.build(params[:article])`会报错，build不能这样用
    5. !：new!, create!, build!与new, create, build的区别是带!的方法会执行validate，如果验证失败会抛出导常。
    6. save是实例方法，而create, build, new是模型类的类方法

33. validate使用：

          include ActiveModel::Validations
          #这里先定义一些有效性判断
          validate :verify_original_password
          validates_presence_of :original_password, :new_password
          validates_confirmation_of :new_password
          validates_length_of :new_password, minimum: 6

          def submit(params)
            self.original_password = params[:original_password]
            self.new_password = params[:new_password]
            self.new_password_confirmation = params[:new_password_confirmation]
            #这里valid?方法会去判断上面所有的validate是否全部有效
            if valid?
              @user.password = new_password
              @user.save!
              true
            else
              false
            end
          end

34. `gem 'papre_trail', github: 'airblade/papre_trail', branch: 'rails4'`


