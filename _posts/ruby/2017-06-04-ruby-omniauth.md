---
layout: post
comments: false
categories: "Ruby"
date: 2017-6-4 00:00:54
title: OmniAuth库
---

<div id="toc"></div>

在介绍Ruby的OmniAuth库之前，有必要提OAuth（开放授权）这个开放标准，OAuth允许用户授权第三方移动应用访问他们存储在另外的服务提供者上的信息，而不需要将用户名和密码提供给第三方移动应用或分享他们数据的所有内容。

这篇文章不会对OAuth标准做详细介绍，如果感兴趣，可以查看[OAuth2.0简介](http://wiki.open.qq.com/wiki/mobile/OAuth2.0%E7%AE%80%E4%BB%8B)。

那OmniAuth库是什么呢，来个官方的英文定义：

```
OmniAuth is a library that standardizes multi-provider authentication for web applications.
```

这里的`provider`可以是facebook, github, qq, weibo, wechat，也可以是自定义的一个provider。一般来说，当你的应用中引入了用户`OmniAuth`并配置了相应的provider后，只需要访问`/auth/:provider`, OmniAuth就可以根据指定的策略去做验证了，譬如打开一个QQ登陆的页面。

但是用户登陆后，下一步是做什么呢？OmniAuth提供了一个给定的url`/auth/:provider/callback`来作为登陆后的callback。但OmniAuth并没有实现这个接口，需要用户自己定义这个接口，并在routes.rb中定义相应route，如:

```
get '/auth/:provider/callback', to: 'sessions#create'
```

> 问题： 回调URL`/auth/:provider/callback`是否需要在provider侧手动配置?

于是在自定义的SessionsController中我们就从request中获取'omniauth.auth': `request.env['omniauth.auth']`。

omniauth.auth中存储的是uid(unique id), 个人信息等，关于详细的可以存储的信息了参考[Auth Hash Schema wiki page](https://github.com/omniauth/omniauth/wiki/Auth-Hash-Schema)。


> OmniAuth库只是修改了request, 没有其他动作，如并不考虑如何自动的把验证后的omniauth.auth的信息与你项目中的User关联起来，不过你只管拿到信息后去用就可以了。

如果在`provider`端验证失败，OmniAuth将获取response并将请求重定向到`/auth/failure`。通过`params[:message]`可以获取到错误信息。

<br />
### 与Rails做集成

#### 配置provider

  在Rails的`initializers`创建`omniauth`的配置，如`config/initializers/omniauth.rb`，配置内容：

```
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :developer unless Rails.env.production?
  provider :twitter, ENV['TWITTER_KEY'], ENV['TWITTER_SECRET']
end
```

这里的`TWITTER_KEY`和`TWITTER_SECRET`是需要手动去twitter申请的。

另这里的配置相对简单，但是实际上不用的`provider`可以配置不同的options，可以参考provider提供的文档，目前omniauth库支持的这些[provider的列表](https://github.com/omniauth/omniauth/wiki/List-of-Strategies)。

如[twitter]https://github.com/arunagw/omniauth-twitter 可以配置为:

```
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :twitter, "API_KEY", "API_SECRET",
    {
      :secure_image_url => 'true',
      :image_size => 'original',
      :authorize_params => {
        :force_login => 'true',
        :lang => 'pt'
      }
    }
end
```

#### 配置session

如果在Rails API中使用OmniAuth, 必须选用如下任一中间件：

- ActionDispatch::Session::CacheStore

- ActionDispatch::Session::CookieStore

- ActionDispatch::Session::MemCacheStore

这些中间件已经默认被Rails加载了，但是这里可能会遇到的一个问题是，session options不其作用，复现此问题的配置是：

- 只是在initializer中增加session_store.rb，设置session相关参数

- 在application.rb中增加use ActionDispatch::Session::CookieStore

问题的原因是Rails在`use ActionDispatch::Session::CookieStore`时，`session_options`将被传递进应用。所以上述配置下，Session功能虽然正常，但是session_options（如session key将默认使用_session_id)不会生效，解决办法是：

- 在application.rb中中间件引用之前，设置相关options，如

```
config.session_store :cookie_store, key: '_interslice_session'
config.middleware.use ActionDispatch::Cookies # Required for all session management
config.middleware.use ActionDispatch::Session::CookieStore, config.session_options
```


#### 设置logger

默认OmniAuth打印log到STDOUT，可以配置其为

```
OmniAuth.config.logger = Rails.logger
```

#### Rails API + devise + devise_token_auth

关于使用后devise_token_auth后需要做的配置，可以参考[devise_token_auth#configuration](https://github.com/lynndylanhurley/devise_token_auth#configuration-tldr), 创建相应user表、model、routes配置等。

另初始化devise_token_auth的配置`config/initializers/omniauth.rb`:

```
DeviseTokenAuth.setup do |config|
  config.change_headers_on_each_request = false
  config.omniauth_prefix = "/api/omniauth"
end
```

这里的config.omniauth_prefix配置的是callback地址的前缀，所以callback地址不在是前面提到的`/auth/:provider/callback`，而变成了`/api/omniauth/:provider/callback`。所以你在provider端也需要手动配置这个callback地址，以QQ为例，在[QQ开发平台](http://op.open.qq.com/)上，需要修改应用开发地址（应用实际调用地址）为`http://[YOU_APP_ADDRESS]/omniauth/qq_connect/callback`。

如果验证成功，调用的逻辑会回到DeviseTokenAuth::ApplicationController的omniauth_success方法，此方法的源码如:

```
def omniauth_success
  get_resource_from_auth_hash
  create_token_info
  set_token_on_resource
  create_auth_params

  if resource_class.devise_modules.include?(:confirmable)
    # don't send confirmation email!!!
    @resource.skip_confirmation!
  end

  sign_in(:user, @resource, store: false, bypass: false)

  @resource.save!

  yield @resource if block_given?

  render_data_or_redirect('deliverCredentials', @auth_params.as_json, @resource.as_json)
end
```

`get_resource_from_auth_hash`中从获取的验证回复中`uid`和`provider`参数并在数据库中查找数据或创建一条记录：

```
def get_resource_from_auth_hash
  # find or create user by provider and provider uid
  @resource = resource_class.where({
    uid:      auth_hash['uid'],
    provider: auth_hash['provider']
  }).first_or_initialize

  if @resource.new_record?
    @oauth_registration = true
    set_random_password
  end

  .....

  @resource
end
```

可以看到，其调用了`@resource.save!`，即验证成功后，就在数据库的user表（概念上的user表，可以是你程序中定义的各种model) 中存储`get_resource_from_auth_hash`中创造的数据。

而`yield @resource if block_given?`中可以在自定义`OmniauthCallbacksController`覆盖`omniauth_success`方法时调用

```
super do
....
end
```

<br />
### 如何集成测试

#### 使用OmniAuth自带的方式
设置OmniAuth测试模式：

```
OmniAuth.config.test_mode = true
```

测试模式下OmniAuth将使用Mock的验证过程，发往`/auth/:provider`的请求将被直接重定向到`/auth/:provider/callback`。如果使用了`devise_token_auth`，则是先重定向`/omniauth/:provider/callback`，而后重定向`auth/:provider/callback`。

代码逻辑将走到`OmniauthCallbacksController`中。



如何Mock验证过程成功的response呢，对OmniAuth.config.mock_auth进行赋值即可。

```
OmniAuth.config.mock_auth[:twitter] = OmniAuth::AuthHash.new({
  :provider => 'twitter',
  :uid => '123545'
  # etc.
})
```

而Mock验证过程失败的response：

```
OmniAuth.config.mock_auth[:twitter] = :invalid_credentials
```

发往`/auth/:provider`的请求将重定向到`/auth/failure?message=invalid_credentials`。


### 参考资料

- [omniauth](https://github.com/omniauth/omniauth)

- [devise_token_auth#omniauth-authentication](https://github.com/lynndylanhurley/devise_token_auth#omniauth-authentication)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
