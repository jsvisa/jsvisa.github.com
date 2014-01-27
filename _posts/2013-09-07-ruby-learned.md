---
layout: post
title: "Ruby 常用语法学习"
description: "Ruby 学习记录"
category: "Ruby Rails"
tags: [Rails, Ruby]

---

1. `nil` 是空对象

    ```ruby
    nil.to_s #=> ''
    nil.nil? #=> true
    ''.nil?  #=> false
    ```
2. Array常用方法 `sort, reverse, shuffle`

    ```ruby
    a.push(6)                       #=> 向数组尾部插入数据6, 等价于 `a<<6`
    a.join(',')                     #=> 串联成一个字符串
    a = %w[foo bar baz]               #=> 创建一个字符串数组
    %w[a b c].map { |i| i.upcase }  #=> map方法返回数组或Range的每个元素执行代码后的结果
    ```
3. inspect 方法返回被调用对象的字符串字面量表达形式

    ```ruby
    flash = { success: "It worked" }
    flash.each do |key, value|
        puts "#{key.inspect}" value "#{value.inspect}"
        #=> :success value "It worked"
        puts "#{key}" value "#{value}"
        #=> success value It worked
    end
    ```
    Ref:  `p` 方法输出对象字面量表示方法:

  `p :name #=> :name` `p "name" #=> "name"`
4. 设置不存在的Hash键时, 返回0而非nil
5. Controller 中定义的实例变量自动在视图中可用，在各方法之间传递参数
6. `user = User.create()` 等价于 `user = User.new(); user.save`;
   `create` 的反操作为 `destroy`, `destroy` 销毁的对象还存在于内存中。
7. **Active Record**

  a. 用到的方法

      ```ruby
      User.find(id) #find方法若找不到则抛出异常
      #find_by方法若找不到则返回nil
      User.find_by_email("example@mail.com")
      User.find_by(email: "example@mail.com")
      User.first  User.last   User.all
      User.all(limit: 6) #前6个
      user.reload  #重新加载当前数据库中的数据
      user.update_attribute(:name, "Wan") #更新属性后都会执行save操作
      user.error.full_messages #查看错误信息
      ```
  b. 回调函数 `before_`，`before_action` 是回调动作,一般用于控件中.
     若不指明限制，将对所有方法都起作用；
     `before_save`, `before_create` 和 `before_update` 进行数据操作,
     一般用于模型使用；
 c. 单复数：

    ```ruby
    include ActionView::Helpers::TextHelper
    pluralize(1, "error")
    => "1 error"
    pluralize(2, "error")
    => "2 errors"
    ```
8. `slice(*keys)` Slice a hash to include only the given keys.
    This is useful for limiting an options hash to valid keys before passing to a method.
    If you have an array of keys you want to limit to, you should splat them:

    ```ruby
    2.0.0p247 :006 > hash = { a: "a", b: "b", c: "c" }
    => {:a=>"a", :b=>"b", :c=>"c"}
    2.0.0p247 :007 > hash.slice(:a, :b)
     => {:a=>"a", :b=>"b"}
    2.0.0p247 :008 > limit = [:a, :b]
     => [:a, :b]
    2.0.0p247 :009 > hash.slice(limit)
     => {}
    2.0.0p247 :010 > hash.slice(*limit)
     => {:a=>"a", :b=>"b"}
    ```

