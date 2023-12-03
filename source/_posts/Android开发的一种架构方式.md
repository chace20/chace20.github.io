---
author: chace
comments: true
date: 2016-01-04 13:47:50+00:00
layout: post
slug: a-framework-of-android-development
title: Android开发的一种架构方式
tags:
- Android
---

本文谈谈博主在实际项目开发的过程中，对代码组织的一种方式。

## 问题的提出

Android系统本身已经给了我们一个MVC。

其中model是业务逻辑，应该是纯Java实现，与平台无关。view一般指xml和自定义view。但是最后的controller应该归于activity，可是写过一段时间代码后我们就会发现，activity承担的责任很多，既要负责网络请求、缓存处理、数据显示，又要处理来自view的事件，代码杂乱不堪，大量的代码积累在activity里面，无论是易读性还是可维护性都大打折扣。

想到这里，controller是不是可以继续分层呢？

## IO层和UI层

博主经过一番思考以及比较后，将原本的controller分成了UI层和IO层，IO层负责网络请求，UI层负责数据展示，两者通过接口回调实现数据传递。

这样做的好处是不言自明的。首先，activity的代码行数可以大幅度减少，代码变得整齐有序。其次，网络请求分出来后实现了数据展示的解耦，事实上，UI层可以完全不管IO层是怎么实现的，符合面向对象的开放-封闭原则。博主在开发中使用xUtils作为网络请求库，UI/IO解耦后，IO层换一种网络请求库对UI层没有任何影响。

## 工程结构

![IO和UI工程结构](/img/2016/01/IO和UI工程结构.jpeg)

## 代码实现

*Talking is cheap, show me the code.*

接下来以一个显示用户信息的页面为例，说明IO/UI具体是怎么实现的。

![UML图](/img/2016/01/Main.jpg)

IOCallback接口定义了网络请求不同阶段和结果处理的方法，其中重载的onSuccess是为了区分返回的List数据和Object数据，使用泛型提高可扩展性，也可以使用Map类型。

IOHandler作为一个handler(处理器，不要理解为Android传递信息的Handler类)父类，在构造函数中初始化ACache，ACache是一个开源库，用来实现缓存。

UserHandler继承自IOHandler，实现具体的handle功能，此处是实现userDetail的处理。

SettingActivity属于UI层，调用UserHandler获取userDetail。其中的getUserDetail方法里面完成调用UserHandler的getUserDetail方法，同时实现IOCallback接口，在回调里拿到数据并显示。

说了那么多，最后上代码吧。

IOCallback：
```
    /**
     * 实现获取数据的回调接口，包括从缓存获取和从服务器获取
     * @link IOHandler
     * @param <T>
     */
    public interface IOCallback<T> {
        /**
         * 开始获取数据
         */
        void onStart();
    
        /**
         * 成功获取数据，数据为list形式
         * @param result
         */
        void onSuccess(List<T> result);
    
        /**
         * 成功获取数据，数据为object形式
         * @param result
         */
        void onSuccess(T result) throws UnsupportedEncodingException;
    
        /**
         * 获取数据失败，一般为网络错误
         * @param error
         */
        void onFailure(String error);
    }
```

IOHandler：
```
    /**
     * IOHandler用于处理数据的获取，在内部调用IOCallback将数据返回给UI层
     */
    public class IOHandler {
        /**
         * Acache对象
         */
        protected ACache cache;
    
        /**
         * 构造函数，初始化Acache
         * @param context
         */
        public IOHandler(Context context){
            this.cache = ACache.get(context);
        }
    }
```

具体的UserHandler：
```
    public class UserHandler extends IOHandler{
    
        public UserHandler(Context context) {
            super(context);
        }
    
       public void getUserDetail(RequestParams params, IOCallback ioCallback) {
            final IOCallback callback = ioCallback;
            String userDetailJson = cache.getAsString(CacheConfig.Key.USER_DETAIL);
            //优先从缓存中读取
            if(userDetailJson!=null){
                try {
                    callback.onSuccess(GsonUtil.getMap(userDetailJson));
                } catch (UnsupportedEncodingException e) {
                    e.printStackTrace();
                }
            }
    
            HttpUtils http = new HttpUtils();
            http.send(HttpRequest.HttpMethod.GET,
                    Api.Users.getUserDetail,
                    params,
                    new RequestCallBack<String>() {
                        @Override
                        public void onStart() {
                            callback.onStart();
                        }
    
                        @Override
                        public void onSuccess(ResponseInfo<String> responseInfo) {
                            Map<String, Object> map = GsonUtil.getMap(responseInfo.result);
                            if(!map.containsKey("code")){
                                cache.put(CacheConfig.Key.USER_DETAIL,responseInfo.result, CacheConfig.Time.USER_DETAIL);
                            }
                            try {
                                callback.onSuccess(map);
                            } catch (UnsupportedEncodingException e) {
                                e.printStackTrace();
                            }
                        }
    
                        @Override
                        public void onFailure(HttpException e, String s) {
                            callback.onFailure(s);
                        }
                    });
        }
    }
```

最后是UI层SettingActivity的调用：
```    
    public class SettingActivity extends BaseActivity {
        private LoadingDialog dialog;
        private UserHandler userHandler;
    
        private TextView unameText;
        private TextView accountText;
    
    
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            unameText = (TextView) findViewById(R.id.unameText);
            accountText = (TextView) findViewById(R.id.accountText);
            getUserDetail();
        }
    
        private void getUserDetail(){
            RequestParams params = new RequestParams();
            params.addQueryStringParameter("token", Constants.token);
            userHandler.getUserDetail(params, new IOCallback<Map<String, Object>>() {
                @Override
                public void onStart() {
    
                }
    
                @Override
                public void onSuccess(List<Map<String, Object>> result) {
    
                }
    
                @Override
                public void onSuccess(Map<String, Object> result) {
                    if (result.containsKey("code")) {
                        Toast.makeText(SettingActivity.this, "getUserDetail error", Toast.LENGTH_SHORT).show();
                    } else {
                        String username = result.get("username").toString();
                        String remark = result.get("remark").toString();
                        unameText.setText(username);
                        accountText.setText(remark);
                    }
                }
    
                @Override
                public void onFailure(String error) {
                    Toast.makeText(SettingActivity.this, "Internet error", Toast.LENGTH_SHORT).show();
                }
            });
        }    
    }
```
