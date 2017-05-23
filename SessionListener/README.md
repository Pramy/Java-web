1. **了解如何使用HttpSessionListener监听session的销毁。**

2. **了解如何使用HttpSessionBindingListener监听session的销毁。**


---
## 一.使用HttpSessionListener编写一个OnlineUserListener。

```
package anni;

import java.util.List;
import javax.servlet.ServletContext;
import javax.servlet.http.HttpSession;
import javax.servlet.http.HttpSessionListener;
import javax.servlet.http.HttpSessionEvent;

public class OnlineUserListener implements HttpSessionListener {

    public void sessionCreated(HttpSessionEvent event) {
    }

    public void sessionDestroyed(HttpSessionEvent event) {
        HttpSession session = event.getSession();
        ServletContext application = session.getServletContext();

        // 取得登录的用户名
        String username = (String) session.getAttribute("username");

        // 从在线列表中删除用户名
        List onlineUserList = (List) application.getAttribute("onlineUserList");
        onlineUserList.remove(username);

        System.out.println(username + "超时退出。");
    }

}
```
OnlineUserListener 实现了HttpSessionListener定义的两个方法：sessionCreated()和sessionDestroyed()。这两个方法可以监听到当前应用中session的创建和销毁情况。我们这里只用到sessionDestroyed()在session销毁时进行操作就可以。

从HttpSessionEvent中获得即将销毁的session，得到session中的用户名，并从在线列表中删除。最后一句向console打印一条信息，提示操作成功，这只是为了调试用，正常运行时删除即可

#### 为了让监听器发挥作用，我们将它添加到web.xml中：

```
<listener>
<listener-class>anni.OnlineUserListener</listener-class>
</listener>
```

## 以下两种情况下就会发生sessionDestoryed（会话销毁）事件：
1. #### 执行session.invalidate()方法时。
    既然LogoutServlet.java中执行session.invalidate()时，会触发sessionDestory()从在线用户列表中清除当前用户，我们就不必在LogoutServlet.java中对在线列表进行操作了，所以LogoutServlet.java的内容现在是这样。
```
public void doGet(HttpServletRequest request,HttpServletResponse response)
          throws ServletException, IOException {
          // 销毁session
          request.getSession().invalidate();
          // 成功
          response.sendRedirect("index.jsp");
      }
```
2. #### 如果用户长时间没有访问服务器，超过了会话最大超时时间，服务器就会自动销毁超时的session。
### 二.使用HttpSessionBindingListener
HttpSessionBindingListener虽然叫做监听器，但使用方法与HttpSessionListener完全不同。我们实际看一下它是如何使用的。

我们的OnlineUserBindingListener实现了HttpSessionBindingListener接口，接口中共定义了两个方法：
## 1. valueBound()  :数据绑定
  所谓对session进行数据绑定，就是调用session.setAttribute()把HttpSessionBindingListener保存进session中。我们在LoginServlet.java中进行这一步:
  ```
  // 把用户名放入在线列表
session.setAttribute("onlineUserBindingListener", new OnlineUserBindingListener(username));
        
  ```
      
```
public void valueBound(HttpSessionBindingEvent event) {
    HttpSession session = event.getSession();
    ServletContext application = session.getServletContext();

    // 把用户名放入在线列表
    List onlineUserList = (List) application.getAttribute("onlineUserList");
    // 第一次使用前，需要初始化
    if (onlineUserList == null) {
        onlineUserList = new ArrayList();
        application.setAttribute("onlineUserList", onlineUserList);
    }
    onlineUserList.add(this.username);
}
```
#### username已经通过构造方法传递给listener，在数据绑定时，可以直接把它放入用户列表。
这就是HttpSessionBindingListener和HttpSessionListener之间的最大区别：HttpSessionListener只需要设置到web.xml中就可以监听整个应用中的所有session。 

HttpSessionBindingListener必须实例化后放入某一个session中，才可以进行监听。

从监听范围上比较，HttpSessionListener设置一次就可以监听所有session，HttpSessionBindingListener通常都是一对一的。

正是这种区别成就了HttpSessionBindingListener的优势，我们可以让每个listener对应一个username，这样就不需要每次再去session中读取username，进一步可以将所有操作在

线列表的代码都移入listener，更容易维护。


## 2. valueUnbound()  :取消绑定
   ```
   public void valueUnbound(HttpSessionBindingEvent event) {
    HttpSession session = event.getSession();
    ServletContext application = session.getServletContext();

    // 从在线列表中删除用户名
    List onlineUserList = (List) application.getAttribute("onlineUserList");
    onlineUserList.remove(this.username);

    System.out.println(this.username + "退出。");
}
   ```
#### 这里可以直接使用listener的username操作在线列表，不必再去担心session中是否存在username。

valueUnbound的触发条件是以下三种情况：

1. **执行session.invalidate()时。**
2. **session超时，自动销毁时。**
3. **执行session.setAttribute("onlineUserListener","其他对象");
或session.removeAttribute("onlineUserListener");将listener从session中删除时**

#### 因此，只要不将listener从session中删除，就可以监听到session的销毁。


   
      


        

