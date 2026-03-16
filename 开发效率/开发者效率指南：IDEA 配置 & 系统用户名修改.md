


# 💻 开发者效率指南：IDEA 配置 & 系统用户名修改

## 目录
-[第一部分：IntelliJ IDEA 注释与版权配置](#第一部分intellij-idea-注释与版权配置)
- [1. 新建文件头注释 (File Header)](#1-新建文件头注释-file-header)
- [2. 方法注释模板 (Live Templates)](#2-方法注释模板-live-templates)
- [3. 类注释快捷键配置](#3-类注释快捷键配置)
- [4. 版权信息 (Copyright) 配置](#4-版权信息-copyright-配置)
- [第二部分：系统电脑用户名修改指南](#第二部分系统电脑用户名修改指南)
  -[1. 仅修改显示名称（安全）](#1-仅修改显示名称安全)
  -[2. 修改用户文件夹名称（开发者必看）](#2-修改用户文件夹名称开发者必看)

---

## 第一部分：IntelliJ IDEA 注释与版权配置

### 1. 新建文件头注释 (File Header)
用于在新建 Java 类或接口时，自动在文件顶部生成的注释。

*   **配置路径**：`Settings` -> `Editor` -> `File and Code Templates` -> `Includes` -> **`File Header`**
*   **推荐模板**：
    ```java
    /**
     * @description TODO
     * @author ${USER}
     * @date ${DATE} ${TIME}
     * @version 1.0
     */
    ```
    *(注：`${USER}`为系统用户名，也可直接写死你的名字)*

### 2. 方法注释模板 (Live Templates)
用于通过输入 `/**` + `Enter`（或 `Tab`）快速生成带参数和返回值的方法注释。

*   **配置路径**：`Settings` -> `Editor` -> `Live Templates`
*   **操作步骤**：
    1. 新建一个 Template Group（如 `MyCustom`）。
    2. 在该组下新建一个 Live Template。
    3. **Abbreviation (缩写)**：填写 `*` （仅输入一个星号）。
    4. **Template text (模板内容)**（注意首行没有 `/`，且需保持缩进）：
       ```java
       *
        * @description $description$
        * @author $user$
        * @date $date$ $time$
        $params$
        * @return $return$
        */
       ```
    5. **Applicable contexts**：点击底部 Define/Change，勾选 `Java`。
    6. 点击 **Edit variables**，按以下表格配置：

| 变量名 | Expression (表达式) | 备注 |
| :--- | :--- | :--- |
| `user` | `user()` | 如果系统名不对可写死固定名称 |
| `date` | `date()` | |
| `time` | `time()` | |
| `return` | `methodReturnType()` | |
| **`params`** | **复制下方 Groovy 脚本填入** | 将参数列表拆分为多行 `@param` |

**params 专属 Groovy 脚本**（直接填入 Expression）：
```groovy
groovyScript("def result=''; def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList(); for(i = 0; i < params.size(); i++) {if(params[i] == '') return result;if(i==0) result += '* @param ' + params[i] + ((i < params.size() - 1) ? '\\n' : '');else result += ' * @param ' + params[i] + ((i < params.size() - 1) ? '\\n' : '')}; return result", methodParameters())
```

### 3. 类注释快捷键配置
如果你想给**已经存在的类**快速添加类注释，推荐使用 `Live Templates` 简写触发。

*   **配置方式**：
    1. 在 `Live Templates` 中新建模板。
    2. **缩写 (Abbreviation)**：设为 `cc` (Class Comment的缩写)。
    3. **模板内容**：
       ```java
       /**
        * @description $description$
        * @author $user$
        * @date $date$
        */
       ```
    4. **适用范围**：勾选 `Java` -> `Declaration`。
*   **使用方法**：在类名上方输入 `cc`，按 `Tab` 键即可秒出注释。
*   *(拓展：若希望使用 `Ctrl+Alt+C` 此类物理组合键，需通过 `Edit -> Macros -> Start Macro Recording` 录制输入 `cc` + `Tab` 的过程，并在 Keymap 中为该宏绑定快捷键。)*

### 4. 版权信息 (Copyright) 配置
用于在代码文件最顶端（`package` 上方）统一声明公司版权。

1.  **新建版权模板**：
    *   `Settings` -> `Editor` -> `Copyright` -> `Copyright Profiles`
    *   点击 `+` 创建新 Profile（如 `MyCompany`），填入内容：
        ```text
        Copyright (c) $today.year. Your Company Name. All rights reserved.
        ```
2.  **设置作用域（关键）**：
    *   回到上一级 `Editor` -> `Copyright`。
    *   在 `Default project copyright` 下拉框中，选择你刚建的 `MyCompany`。
3.  **格式化调整（可选）**：
    *   `Editor` -> `Copyright` -> `Formatting` -> `Java`。
    *   勾选 `Add blank line after`，使版权头与包名间留空行。
4.  **如何应用**：
    *   **新文件**：自动生成。
    *   **旧文件**：双击 `Shift` 搜索 `Update Copyright`，选择 `Whole project` 批量更新全项目。

---

## 第二部分：系统电脑用户名修改指南

### 1. 仅修改显示名称（安全）
这只改变锁屏、开始菜单显示的昵称，不影响底层文件结构。

*   **Windows (Microsoft 账号登录)**：
    前往 [account.microsoft.com/profile](https://account.microsoft.com/profile)，登录后点击“编辑名称”，重启电脑后联网同步生效。
*   **Windows (本地账号登录)**：
    `Win + R` 输入 `control` 打开控制面板 -> `用户账户` -> `更改账户名称`。
*   **macOS**：
    `系统设置` -> `用户与群组` -> 右键当前用户选 `高级选项` -> 修改 `全名 (Full name)`。（**警告：绝不要改“账户名称”或“个人文件夹”字段**）。

### 2. 修改用户文件夹名称（开发者必看）
**场景**：开发时因为 `C:\Users\中文名` 导致 IDEA、Maven、Python 报错，需要改成纯英文路径。
**⚠️ 警告**：直接重命名 `C:\Users` 下的文件夹会导致系统崩溃或应用丢失。

**✅ 唯一推荐的安全解决方案：新建管理员账号**
这是开发者的最彻底做法：
1.  **新建英文账号**：
    `Win + I` 打开设置 -> `账户` -> `其他用户` -> `添加其他人` -> `我没有这个人的登录信息` -> `添加一个没有 Microsoft 账户的用户`。
    **注：用户名填纯英文（如 `dev`、`kevin`），切勿带空格。**
2.  **赋予管理员权限**：
    点击刚建好的账号 -> `更改账户类型` -> `管理员`。
3.  **数据迁移与交接**：
    *   注销当前账号，登录新英文账号。
    *   将旧账号 `C:\Users\旧名字` 下的 `桌面(Desktop)`、`文档(Documents)` 等个人文件拷贝到新账号下。
    *   在新账号下重新配置开发环境（路径已变为纯英文）。
4.  **清理（可选）**：确认一切运转正常后，可选择在系统设置中删除原来的旧账号以释放空间。