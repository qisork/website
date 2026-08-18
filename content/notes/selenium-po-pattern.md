---
title: Selenium 的 PO 模式
date: 2025-07-31T09:12:23+08:00
tags:
  - selenium
  - testing
  - 笔记
draft: false
description: 本文记录了我对 Selenium 的 PO 模式一些理解，其中包含 PO 模式是什么，以及相应的示例。
summary: 本文记录了我对 Selenium 的 PO 模式一些理解，其中包含 PO 模式是什么，以及相应的示例。
---

> 本文记录了我对 Selenium 的 PO 模式一些理解，其中包含 PO 模式是什么，以及相应的示例。

在我第一眼看到 PO 模式（Page Object Model）时，心里就有两个疑惑：

- 这是什么东东？
- 我又该如何使用？

带着这两个疑惑，去查找资料了解到，它是一种设计模式，将定位和操作页面元素的代码与测试代码相分离，以便提高测试的可维护性和复用性。

也就是说，对页面元素的定位和操作进行封装，然后提供一个向上的统一接口，再进行调用。

这么做有一个好处，如果页面元素发生变化（如标签 `id` 改变，`html` 结构变化等），只需要对先前封装的代码进行调整，而无需改变测试代码。

这里有个一例子，是没有使用 PO 模式的代码：

```python
import pytest
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

@pytest.fixture  
def driver():  
    driver = webdriver.Edge()  
    driver.get("https://example.com/login")  
    yield driver  
    driver.quit()

def test_login_without_po(driver):
	# 直接在测试用例中定位和操作元素
	# 等待用户名输入框加载完成
	username_input = WebDriverWait(driver, 10).until(
		EC.presence_of_element_located((By.ID, "username"))
	)
	username_input.send_keys("test_user")
	
	# 等待密码输入框加载完成
	password_input = WebDriverWait(driver, 10).until(
		EC.presence_of_element_located((By.ID, "password"))
	)
	password_input.send_keys("test_password")
	
	# 等待登录按钮加载完成并点击
	login_button = WebDriverWait(driver, 10).until(
		EC.element_to_be_clickable((By.ID, "login-button"))
	)
	login_button.click()
	
	# 验证登录成功后的欢迎消息
	login_message = WebDriverWait(driver, 10).until(
		EC.visibility_of_element_located((By.ID, "login_message"))
	)
	assert "Welcome" in login_message.text
```

可以看到大量的元素操作细节和测试逻辑混合在一起，不仅违背了**单一职责原则**，还导致大量重复代码出现，而它们之间的区别仅仅是页面元素不同。

试想一下，目前只是针对登录功能的测试，所以代码量看起来比较小，但如果要测试整个网站的功能（如注册、注销、修改个人资料等），那同样的页面元素的定位和操作岂不是每个测试中都要再重复出现一次。

并且，在开发过程中网页结构是会不断变化的，每一次变化，我们都需要重新调整测试代码，一两个还好，但架不住量大啊！

我自诩一位懒人，自然要尽可能的偷懒喽。

## PO 模式的核心概念

写到这里，必然要介绍一番 PO 模式的核心概念，以便在下面的示例中有更好的理解。

- Page object（页面对象）
	- 将每个网页抽象为一个类，页面中的元素和操作封装为类的属性和方法。
	- 例如，登录页面可抽象为 `LoginPage` 类。
- 将页面操作与测试用例分离的关注点
	- **页面对象**：负责元素定位、交互操作（如点击、输入）。
	- **测试用例**：专注于业务逻辑，调用页面对象的方法完成测试。

## PO 模式的实现示例

现在开始实操。根据上面所述，将页面操作和测试用例进行分离，那么可以将项目划分为：

- **页面对象层**：每个页面一个类，封装元素和操作。
- **测试用例层**：调用页面对象完成测试。

项目结构如下：
```txt
my_project/
├── pages/
│   ├── page_login.py
│   └── ...
└── tests/
    ├── test_login.py
    └── ...
```

### 定义 `page_login.py`

假设我们要测试一个登录界面，可以创建一个 `LoginPage` 类，封装“登录”操作。

```python
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class LoginPage:
    def __init__(self, driver):
        """初始化 LoginPage 实例。
        
        参数:
            driver (WebDriver): 传入的 Selenium WebDriver 实例，
                              用于与浏览器进行交互。
        """
        self.__driver = driver
        self.__username_input = (By.ID, "username")
        self.__password_input = (By.ID, "password")
        self.__login_button = (By.ID, "login_button")
        self.__login_message = (By.ID, "login_message")

    def login(self, username, password):
        """登录系统。

        参数:
            username (str): 用户名
            password (str): 密码
        """
        self.__enter_username(username)
        self.__enter_password(password)
        self.__click_login()

    def get_login_message(self):
        """获取登录消息文本。

        返回:
            str: 登录消息的文本内容。
        """
        return WebDriverWait(self.__driver, 10).until(
            EC.visibility_of_element_located(self.__login_message)
        ).text

    def __enter_username(self, username):
        """在用户名输入框中输入指定的用户名。

        参数:
            username (str): 要输入的用户名
        """
        WebDriverWait(self.__driver, 10).until(
            EC.visibility_of_element_located(self.__username_input)
        ).send_keys(username)

    def __enter_password(self, password):
        """在密码输入框中输入指定的密码。

        参数:
            password (str): 要输入的密码
        """
        WebDriverWait(self.__driver, 10).until(
            EC.visibility_of_element_located(self.__password_input)
        ).send_keys(password)

    def __click_login(self):
        """等待登录按钮可点击，并执行点击操作。"""
        WebDriverWait(self.__driver, 10).until(
            EC.element_to_be_clickable(self.__login_button)
        ).click()
```

### 定义 `test_login.py`

现在创建 `test_login.py` 来测试登录功能。

```python
import pytest
from selenium import webdriver
from pages.page_login import LoginPage

@pytest.fixture
def driver():
    """创建并配置一个Edge WebDriver实例，用于测试登录功能。

    返回:
        WebDriver: 配置好的Edge WebDriver实例
    """
    driver = webdriver.Edge()
    driver.get("https://example.com/login")
    yield driver
    driver.quit()

def test_login(driver):
    """测试登录功能。

    参数:
        driver (WebDriver): Selenium WebDriver实例，用于浏览器自动化操作。
    """
    login_page = LoginPage(driver)
    login_page.login("test_user", "test_password")
    assert "Welcome" in login_page.get_login_message(), "登录失败，未找到欢迎信息"
```

## 进一步封装：`base.py`

到这里，这个例子完善了吗？不，关于页面元素的操作还可以进一步封装，将共有的操作提出出来。

这么说可能觉得不明所以，心里会有一个疑问：“是可以再进一步封装，但为什么要呢？”

这个答案在于，为了可维护性。试想一下，假如在一个程序中，我们将要频繁的使用 `"passwd"` 这个字符串，是使用一个变量保存它，还是直接使用？

当然是选择定义一个变量保存，这样如果日后有需要，就可以更改一处，得到全局自动获取新内容。

在这里，也是一样的，将相同的操作抽离出来，如果以后有更好的实现方式，就可以只更改这一处，而无需更改其他代码。

所以我们再添加一个基础层，用来存放通用操作。

更新后的项目结构如下：
```txt
my_project/
├── bases/        # 新添加
│   ├── base.py   # 存放通用操作
│   └── ...
├── pages/
│   ├── page_login.py
│   └── ...
└── tests/
    ├── test_login.py
    └── ...
```

### 定义 `base.py`

定义 `Base` 类，存放元素查找，和一些常见的操作方法（如点击、输入文本等）。

```python
from selenium.webdriver.support.wait import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class Base:
    def __init__(self, driver):
        """初始化Base类。

        参数:
            driver (WebDriver): WebDriver实例，用于与浏览器进行交互。
        """
        self.driver = driver

    def base_find_element(self, locator, timeout=10):
        """等待指定的元素在页面上变得可见。

        参数:
            locator (tuple): 用于定位元素的元组，通常由By和定位值组成。
            timeout (int, optional): 等待元素的最大时间（秒）。默认为10秒。

        返回:
            WebElement: 等待到的可见元素。
        """
        return WebDriverWait(self.driver, timeout=timeout).until(
            EC.visibility_of_element_located(locator)
        )

    def base_click(self, locator):
        """执行元素点击操作。

        参数:
            locator (tuple): 定位元素的元组，包含定位方式和定位值。
        """
        self.base_find_element(locator).click()

    def base_send_keys(self, locator, text):
        """在定位到的元素中输入文本。

        参数:
            locator (tuple): 定位元素的元组，通常包含定位方式和定位值。
            text (str): 需要输入的文本内容。
        """
        element = self.base_find_element(locator)
        element.clear()
        element.send_keys(text)
```

### 修改 `page_login.py`

目前只需要修改 `page_login.py`，而无需动测试用例，因为我们已经为测试用例提供了统一的调用接口，只要我们不改动这个，就可以。封装的魅力。

```python
from selenium.webdriver.common.by import By
from bases.base import Base

class LoginPage:
    def __init__(self, driver):
        """初始化 LoginPage 实例。
        
        参数:
            driver (WebDriver): 传入的 Selenium WebDriver 实例，
                              用于与浏览器进行交互。
        """
        self.__base = Base(driver)
        self.__username_input = (By.ID, "username")
        self.__password_input = (By.ID, "password")
        self.__login_button = (By.ID, "login_button")
        self.__login_message = (By.ID, "login_message")

    def login(self, username, password):
        """登录系统

        参数:
            username (str): 用户名
            password (str): 密码
        """
        self.__enter_username(username)
        self.__enter_password(password)
        self.__click_login()

    def get_login_message(self) -> str:
        """获取登录消息文本。

        返回:
            str: 登录消息的文本内容。
        """
        return self.__base.base_find_element(self.__login_message).text

    def __enter_username(self, username):
        """在用户名输入框中输入指定的用户名。

        参数:
            username (str): 要输入的用户名
        """
        self.__base.base_send_keys(self.__username_input, username)

    def __enter_password(self, password):
        """在密码输入框中输入指定的密码。

        参数:
            password (str): 要输入的密码
        """
        self.__base.base_send_keys(self.__password_input, password)

    def __click_login(self):
        """等待登录按钮可点击，并执行点击操作。"""
        self.__base.base_click(self.__login_button)
```

## 总结

PO 模式通过将页面的元素和操作封装为对象，使测试代码更清晰、可维护和可复用。

- **提高代码复用性**：多个测试用例可共享同一页面对象。
- **增强可维护性**：页面变化时只需更新页面对象。
- **提升可读性**：测试用例更接近自然语言，如 `login_page.enter_username("test")`。

根据 PO 模式可以将项目结构划分为三层：

- **基础层**：封装通用方法。
- **页面对象层**：每个页面一个类，封装元素和操作。
- **测试用例层**：调用页面对象完成测试。

