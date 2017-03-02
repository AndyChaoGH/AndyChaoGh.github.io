---
layout: post
title: MVPDagger2Demo
author: arvinljw
---

最开始创建这个Demo主要是为了学习一下Dagger2，结果从中还学习到了MVP设计模式，当然这肯定是最基本的含义，在实际业务中去使用还得多加练习，现在总结一下，我理解到的MVP模式。

# MVPDemo
这是一个简单的MVP＋Dagger2的Demo，在我学习完后一直没有记录，这周末抽空总结一下。之后还会把Dagger2的理解和使用也一起发在这篇文章下。


进入主题：业务很简单，1、保存输入的姓名、性别、年纪用户信息，用户信息都不能为空；2、根据输入的姓名查询用户信息，没有输入时将查询出来全部用户。效果图如下：

![](https://github.com/arvinljw/DaggerDemo/blob/master/screen/1.png)

![](https://github.com/arvinljw/DaggerDemo/blob/master/screen/2.png)


## MVP理解
1、目的：

在我看来MVP模式就是为了实现解耦，避免出现上帝类，让业务功能更加清晰，业务清晰了，就不那么容易出现Bug了，同时维护起来也更加容易。

2、包含的模块：

正所谓MVP，M表示Model，V表示View，P表示Presenter。（注：Xxx表示业务名字）
	
Model会包含三个部分，业务的实体类Entity，业务功能接口IXxxBiz，业务功能的实现XxxBiz;

View包含的两个部分，UI功能接口IXxxView，UI功能的实现就是Activity或者Fragment，当然我觉得Adapter就没有必要再使用这个了，应该这个已经是MVC模式了，已经实现解耦了。

Presenter包含两个部分，View和Biz联系的接口IXxxPresenter，另一个当然就是View和Biz联系的实现。presenter他就是一个主持人，将View和Biz的功能联系起来，这样就把View和Biz解耦了。
	

## 业务分析
上边简单的介绍了一下MVP的理解，现在开始讲解一下业务。

根据业务我们可以分为三个部分，输入内容：姓名、性别、年纪和查询的姓名；操作按钮：保存和查询；显示结果：一个文本框。MVP模式的一大难点就在于业务的抽象、提取工作，这也是我需要练习的地方，接下来就开始提取。

首先从业务上来看，我们能很清晰的知道就是保存用户和查询用户两个功能，用户的信息就只包含了姓名、性别、年龄。所以Model这一块就可以简单的提取一下，实体类就是UserEntity：

```
public class UserEntity {
    private String name;
    private String sex;
    private int age;

    public UserEntity(String name, String sex, int age) {
        this.name = name;
        this.sex = sex;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getSex() {
        return sex;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "UserEntity{" +
                "name='" + name + '\'' +
                ", sex='" + sex + '\'' +
                ", age=" + age +
                '}';
    }
}
```
这里边直接重写了toString方法，方便我们待会直接显示用户信息。

然后时IUserBiz：

```
public interface IUserBiz {
    void saveUser(UserEntity user);

    List<UserEntity> getUserByName(String name);
}
```
这里保存的办法有很多种，例如网络存储、文件存储、本地数据库存储等，这里就简单一点，就不使用这三种了，直接保存到Application中，相当于临时存一下。所以获取也就很容易，根据名字去获取匹配的用户列表即可,逻辑很简单就不解释了。

```
public class App extends Application {
    public static App INSTANCE;
    private List<UserEntity> users = new ArrayList<>();

    @Override
    public void onCreate() {
        super.onCreate();
        INSTANCE = this;
    }

    public List<UserEntity> getUsers() {
        return users;
    }
}
```

```
public class UserBiz implements IUserBiz {

    public UserBiz() {
    }

    @Override
    public void saveUser(UserEntity user) {
        App.INSTANCE.getUsers().add(user);
    }

    @Override
    public List<UserEntity> getUserByName(String name) {
        if (TextUtils.isEmpty(name)) {
            return App.INSTANCE.getUsers();
        }
        List<UserEntity> users = new ArrayList<>();
        for (UserEntity entity : App.INSTANCE.getUsers()) {
            if (entity.getName().equals(name)) {
                users.add(entity);
            }
        }
        return users;
    }
}
```

接下来，看看View模块怎么提取，首先根据业务流程来看，保存用户信息需要得到输入框中的姓名、性别以及年龄，所以很明显我们需要三个get方法来分别获取三个用户信息；然后，点击保存，为了交互友好一些，所以我们应该有提示信息，分别是保存成功或者保存失败的提示信息；接着，根据用户名字查询用户信息并显示查询结果，和保存同理，也需要一个get方法获取输入的姓名，以及查询成功和查询失败。所以我们就能得出IUserView，如下：

```
public interface IUserView {
    String getName();

    String getSex();

    String getAge();

    String getSelectName();

    void saveSuccess(String successMessage);

    void saveFailed(String errorMessage);

    void showUsers(List<UserEntity> users);

    void showUserFailed(String errorMessage);
}
```
其实现也很简单，但是为了实现IUserView的话，需要用到Presenter，所以我们就先看看Presenter时如何去提取以及实现的。

在上边说到Presenter是作为View和Model的纽带，那么它包含的功能又有哪些呢？还是得从业务说起，保存用户信息的时候，我们就是向Application中插入一条用户信息，当然是需要验证信息是否为空；在获取查询时，再向Application中去取就行了。到这里完全可以不用使用Presenter也可以做完了，只不过就没有实现View和Model的解耦而已。为此，我们引入了IUserPresenter，如下：

```
public interface IUserPresenter {
    void saveUser();

    boolean validate();

    void getUserByName();
}
```
实现逻辑也很简单，如下：

```
public class UserPresenter implements IUserPresenter{
    private IUserView iUserView;
    
    private IUserBiz userBiz;

    
    public UserPresenter(IUserView iUserView) {
        this.iUserView = iUserView;
        this.userBiz = new UserBiz();
    }

    @Override
    public void saveUser() {
        if (!validate()) {
            return;
        }

        userBiz.saveUser(new UserEntity(iUserView.getName(), iUserView.getSex(), Integer.valueOf(iUserView.getAge())));
        iUserView.saveSuccess("保存成功!");
    }

    @Override
    public boolean validate() {
        String errorMessage = "";
        if (isEmpty(iUserView.getName())) {
            errorMessage = "姓名不能为空!";
        }
        if (isEmpty(iUserView.getSex())) {
            errorMessage = "性别不能为空!";
        }
        if (isEmpty(iUserView.getAge())) {
            errorMessage = "年纪不能为空!";
        }
        if (!isEmpty(errorMessage)) {
            iUserView.saveFailed(errorMessage);
            return false;
        }
        return true;
    }

    @Override
    public void getUserByName() {
        List<UserEntity> users = userBiz.getUserByName(iUserView.getSelectName());
        if (users == null || users.size() == 0) {
            iUserView.showUserFailed("未查询到结果!");
        }
        iUserView.showUsers(users);
    }

    private boolean isEmpty(String str) {
        return TextUtils.isEmpty(str);
    }
}
```
这样需要将IUserView的实现传进来外边通过Presenter直接调用就可以了，接下来就是IUserView的实现，如下：（其中用到了ButterKnife8.4，具体的使用方式这里就不介绍了，总之也是很好用的一个框架）

```
public class MainActivity extends AppCompatActivity implements IUserView {
    @BindView(R.id.ed_name)
    EditText edName;
    @BindView(R.id.ed_sex)
    EditText edSex;
    @BindView(R.id.ed_age)
    EditText edAge;
    @BindView(R.id.ed_selected_name)
    EditText edSelectedName;

    @BindView(R.id.tv_result)
    TextView tvResult;

    UserPresenter userPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);

        userPresenter = new UserPresenter(this);
    }

    @OnClick({R.id.btn_save, R.id.btn_search})
    void onClick(View v) {
        switch (v.getId()) {
            case R.id.btn_save:
                userPresenter.saveUser();
                break;
            case R.id.btn_search:
                userPresenter.getUserByName();
                break;
        }
    }

    @Override
    public String getName() {
        return edName.getText().toString().trim();
    }

    @Override
    public String getSex() {
        return edSex.getText().toString().trim();
    }

    @Override
    public String getAge() {
        return edAge.getText().toString().trim();
    }

    @Override
    public String getSelectName() {
        return edSelectedName.getText().toString().trim();
    }

    @Override
    public void saveSuccess(String successMessage) {
        showToast(successMessage);
    }

    @Override
    public void saveFailed(String errorMessage) {
        showToast(errorMessage);
    }

    @SuppressWarnings("StringConcatenationInsideStringBufferAppend")
    @Override
    public void showUsers(List<UserEntity> users) {
        StringBuilder sb = new StringBuilder();
        for (UserEntity user : users) {
            sb.append(user.toString() + "\n");
        }
        tvResult.setText(sb.toString());
    }

    @Override
    public void showUserFailed(String errorMessage) {
        showToast(errorMessage);
    }

    private void showToast(String errorMessage) {
        Toast.makeText(this, errorMessage, Toast.LENGTH_SHORT).show();
    }
}

```
布局也很简单，也就不解释了，如下：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context=".uis.activities.MainActivity">

    <EditText
        android:id="@+id/ed_name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="请输入姓名"
        android:inputType="text" />

    <EditText
        android:id="@+id/ed_sex"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="请输入性别"
        android:inputType="text" />

    <EditText
        android:id="@+id/ed_age"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="请输入年纪"
        android:inputType="number" />

    <Button
        android:id="@+id/btn_save"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="保存" />

    <EditText
        android:id="@+id/ed_selected_name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:hint="请输入要查询的用户姓名"
        android:inputType="text" />

    <Button
        android:id="@+id/btn_search"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="查询" />

    <TextView
        android:id="@+id/tv_result"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp" />
</LinearLayout>
```
到这里，所有的业务就写完了，虽然多了好多类，但是实现了View和Model的解耦，对业务也很清晰，最主要是有一种心情舒畅的感觉。因为你要是维护过几千行代码的Activity或者Fragment类的话，你就会觉得这样写真是确实舒服了。



## Thanks

写这篇文章也是在阅读了鸿洋大神的[浅谈 MVP in Android](http://blog.csdn.net/lmj623565791/article/details/46596109)文章之后才有的感觉。

也要感谢其他作者的分享，没有之前的积累也肯定写不出来。

在此表示感谢！


## License
```
Copyright 2016 arvinljw

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

## 源码

**[源码链接](https://github.com/arvinljw/DaggerDemo)**

