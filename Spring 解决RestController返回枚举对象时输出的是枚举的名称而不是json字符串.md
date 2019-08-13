---
title: Spring 解决RestController返回枚举对象时输出的是枚举的名称而不是json字符串
date: 2019-03-16 20:34:51
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/88605184]( https://blog.csdn.net/abc123lzf/article/details/88605184)   
  #### []()举个例子：

 
```
private enum LoginResult {
    SUCCESS(0, "成功登入"),
    USER_NOT_FOUND(1, "未找到用户"),
    INCORRECT_PASS(2, "密码错误"),
    INCORRECT_CODE(3, "验证码错误");

    private final int code;
    private final String msg;

    LoginResult(int code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    @Override
    public int getCode() {
        return this.code;
    }

    @Override
    public String getMsg() {
        return this.msg;
    }
}

@RequestMapping("login")
@ResponseBody
public LoginResult login(HttpSession session) {
	//省略业务逻辑
	return LoginResult.SUCCESS;
}

```
 在上述场景，我们希望前端返回的是这样的Json数据：

 
```
{"code": 0, "msg": "成功登入"}

```
 然而实际上返回的是：

 
```
"SUCCESS"

```
 问题有两种解决方式，如果你使用的是Jackson作为Json的序列化工具，那么你可以在枚举类上加上这个注解：

 
```
@JsonFormat(shape = JsonFormat.Shape.OBJECT)
private enum LoginResult {
	//...
}

```
 问题即可解决。

 如果你希望用FastJson作为序列化工具，有一种解决办法就是创建一个类并继承FastJsonHttpMessageConverter。然后将你的枚举类实现自定义接口，比如ResponseObject：

 
```
public interface ResponseObject {
	int getCode();
	String getMsg();
}

```
 FastJsonHttpMessageConverter是AbstractHttpMessageConverter的子类，我们需要重写它的writeInternal方法：

 
```
import com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter;
import com.jsu.shoppingmall.controller.common.ComplexResponseObject;
import com.jsu.shoppingmall.controller.common.ResponseObject;
import org.springframework.http.HttpInputMessage;
import org.springframework.http.HttpOutputMessage;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.http.converter.HttpMessageNotWritableException;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.io.OutputStream;
import java.nio.charset.Charset;

@Component
public class DefaultFastJsonHttpMessageConverter extends FastJsonHttpMessageConverter {

    private static final Charset charset = Charset.forName("UTF-8");

    public DefaultFastJsonHttpMessageConverter() {
        super();
    }

    @Override
    protected void writeInternal(Object o, HttpOutputMessage httpOutputMessage) throws IOException, HttpMessageNotWritableException {
        if(o instanceof ResponseObject && o instanceof Enum<?>) {
            ResponseObject obj = (ResponseObject) o;
            String text = String.format("{\"code\": %d, \"msg\": \"%s\"}", obj.getCode(), obj.getMsg());
            OutputStream out = httpOutputMessage.getBody();
            byte[] bytes = text.getBytes(charset);
            out.write(bytes);

            return;
        }

        super.writeInternal(o, httpOutputMessage);
    }
}

```
 上述writeInternal方法会判断是不是ResponseObject的枚举实现类，如果是就按照我们指定的步骤将其序列化为Json字符串。  
 如果实在不想定义ResponseObject接口，那么直接通过反射的方式获取也是可以的。

 
#### []()扩展文章

 [https://www.cnblogs.com/page12/p/8166935.html](https://www.cnblogs.com/page12/p/8166935.html)

   
  