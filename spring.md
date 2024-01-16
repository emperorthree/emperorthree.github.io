
# mine-spring [updating]

Ref：[手写Spring - 廖雪峰的官方网站 (liaoxuefeng.com)](https://www.liaoxuefeng.com/wiki/1539348902182944)

参考廖老师的博客，手写自己的spring框架，目的深入了解spring的工作方式及设计原理；



mine-spring项目结构

* parant : 仅依赖管理
* mine-context : 类注解扫描、BeanDefinition、实例化和初始化
* mine-aop : 字节码增强，byteBody生成代理对象；代理对象的创建以及原始对象的属性注入问题
* mine-jdbc : jdbcTemplate和事物注解
* mine-web : DispatherServlet，MVC注解，modelAndView
* mine-boot : 嵌入Tomcat，启动Tomcat服务时自动初始化创建DispatcherServlet和ApplicationContext


<ul>
    {% for post in site.posts %}
        {% if post.category=='spring' %}
        <li>
            <div>
                <span>{{ post.date | date: "%Y-%m-%d" }}</span>
                <a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
            </div>
        </li>
        {% endif %}
    {% endfor %}
</ul> 