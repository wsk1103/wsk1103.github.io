---
title: "使用@ControllerAdvice + @ExceptionHandler 全局处理Controller 层异常"

tags:
  - Java
---

目的：处理异常，并返回特定的内容，防止返回异常的内容到客户端。并且邮件通知到开发，使开发人员能够第一时间知晓。日志一键记录，不需要再额外记录和增加扩展性，维护性。  

邮件通知限制：  

	1. 邮件通知类MailUtil
	2. 同一个邮件标题5分钟内只发一次。
	3. 同一个mail内容（包括主题，收件人，主体，抄送人）同一个小时内只发送一次（同一个小时指15点，16点等）


 @ControllerAdvice + @ExceptionHandler 进行全局的 Controller 层异常处理，只要设计得当，就再也不用在 Controller 层进行 try-catch 了  
 
 一、优缺点  
**优点**：将 Controller 层的异常和数据校验的异常进行统一处理，减少模板代码，减少编码量，提升扩展性和可维护性。  
**缺点**：只能处理 Controller 层未捕获（往外抛）的异常，对于 Interceptor（拦截器）层的异常，Spring 框架层的异常，就无能为力了。
#### 1. 基本使用示例


**基类**
```

import com.alibaba.fastjson.JSONObject;
import org.apache.commons.lang3.exception.ExceptionUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.web.bind.annotation.ResponseStatus;

import javax.servlet.http.HttpServletRequest;

public class BaseGlobalExceptionHandler extends BaseController {

    protected static final Logger logger = LoggerFactory.getLogger(BaseGlobalExceptionHandler.class);
    /**
     * 全局统一异常返回信息
     */
    protected static final String DEFAULT_ERROR_MESSAGE = "系统维护，请稍后访问";

    /**
     * 主要的处理
     *
     * @param req 请求
     * @param e   异常
     * @return 错误返回结果
     * @throws Exception 处理失败
     */
    protected byte[] handleError(HttpServletRequest req, Exception e) throws Exception {
        //必须有注解响应码才会触发，例如500,504等等
        if (AnnotationUtils.findAnnotation(e.getClass(), ResponseStatus.class) != null) {
            throw e;
        }
        String mailMsg = String.format("Request url is  %s , \nand the error is %s", req.getRequestURI(), ExceptionUtils.getStackTrace(e));
        logger.error(mailMsg);
        //发送邮件
        logger.info(MailUtil.sendMail(e.getMessage(), mailMsg));
        return handler();
    }

    /**
     * 默认处理方式
     *
     * @return 错误返回结果
     * @throws Exception 处理失败
     */
    protected byte[] handler() throws Exception {
        JSONObject jsonObject = new JSONObject();
        return getFaildResponse(jsonObject, DEFAULT_ERROR_MESSAGE);
    }
}
```
**具体处理类**
```
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.ResponseStatus;

import javax.servlet.http.HttpServletRequest;

@ControllerAdvice
public class GlobalExceptionHandler extends BaseGlobalExceptionHandler {

    /**
     * 500的所有异常会被这个方法捕获
     *
     * @param req 请求
     * @param e   异常
     * @return 输出
     * @throws Exception 未知异常
     */
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ResponseBody
    public byte[] handleAllError(HttpServletRequest req, Exception e) throws Exception {
        return super.handleError(req, e);
    }

    /**
     * 500的事务异常会被这个方法捕获
     *
     * @param req 请求
     * @param e   异常
     * @return 输出
     * @throws Exception 未知异常
     */
    @ExceptionHandler(BizException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ResponseBody
    public byte[] handleBizError(HttpServletRequest req, Exception e) throws Exception {
        return super.handleError(req, e);
    }

}
```
三、 代码说明  
Logger 进行所有的异常日志记录。  
@ExceptionHandler(BizException.class) 声明了对 BizException 业务异常的处理，并获取该业务异常中的错误提示，构造后返回给客户端。  
@ExceptionHandler(Exception.class) 声明了对 Exception 异常的处理，起到兜底作用，不管 Controller 层执行的代码出现了什么未能考虑到的异常，都返回统一的错误提示给客户端。  
备注：以上 GlobalExceptionHandler 只是返回 Json 给客户端，更大的发挥空间需要按需求情况来做。