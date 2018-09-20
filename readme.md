---
title: Token注解实现前后端安全访问
date: 2018-09-11 03:33:00
tags: 
- SpringBoot
category: 
- SpringBoot
description: Token注解实现前后端安全访问
---
<!-- image url 
https://raw.githubusercontent.com/HealerJean123/HealerJean123.github.io/master/blogImages
　　首行缩进
<font color="red">  </font>
-->

## 前言

## 1、Tocken注解

```
package com.duodian.youhui.admin.config.token.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 校验用户token
 * 可以用于class 或者 method
 * 如果在method和class都有，则以method的为准。
 * @author HealerJean
 */
@Target(value = {ElementType.TYPE,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Token {

    /**
     * 是否校验token
     * @return
     */
    boolean check() default true;
}


```

## 2、用户登录保存Token

```

@ApiOperation(value = "第二步：通过code换取网页授权access_token",
        notes = "第二步：通过code换取网页授权access_token",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE,
        response = ResponseBean.class
)
@ApiImplicitParams({
        @ApiImplicitParam(name = "code", value = " code作为换取access_token的票据，每次用户授权带上的code将不一样，code只能使用一次，5分钟未被使用自动过期。", required =true,paramType = "query", dataType = "string"),
        @ApiImplicitParam(name = "scope", value = "应用授权作用域 snsapi_base,snsapi_userinfo", required =true,paramType = "query", dataType = "string"),
        @ApiImplicitParam(name = "fuWuBusinessNoId", value = "服务号 数据库中存放微信运营者的，主键，服务区号哆趣商城 3： ",paramType = "query", dataType = "int"),
        @ApiImplicitParam(name = "dingYueBusinessNoId", value = "订阅号 数据库中存放微信运营者的，主键，火影情报室 2： ",paramType = "query", dataType = "int"),
        @ApiImplicitParam(name = "userInfoPId", value = "推荐人Id", required =false,paramType = "query", dataType = "long")

})
@GetMapping( value = "youhui/web/authorize/redirect",produces="application/json;charset=utf-8")
@ResponseBody
public ResponseBean authorizeRedirect(String code,Long fuWuBusinessNoId,Long dingYueBusinessNoId,String scope,Long userInfoPId){
    try {
        return  ResponseBean.buildSuccess(userInfoService.loginToken(fuWuBusinessNoId,dingYueBusinessNoId,code,scope,userInfoPId));
    } catch (AppException e) {
        ExceptionLogUtils.log(e, this.getClass());
        return ResponseBean.buildFailure(e.getCode(),e.getMessage());
    } catch (Exception e) {
        ExceptionLogUtils.log(e, this.getClass());
        return ResponseBean.buildFailure(e.getMessage());
    }
}


service中

/**
 * 用户登录返回Token,之后利用拦截器中的toke
 * 获取用户当前上下文中的的id和openId
 * @param fuWuBusinessNoId
 * @param dingYueBusinessNoId
 * @param code
 * @param scope
 * @param userInfoPId
 * @return
 */
@Override
public String loginToken(Long fuWuBusinessNoId,Long dingYueBusinessNoId ,String code,String scope,Long userInfoPId) {
    UserInfo userInfo = weChatWebService.getAccessTokenByCode(fuWuBusinessNoId,dingYueBusinessNoId,code,scope,userInfoPId);
    String token =  setAppToken(userInfo.getId());
    return token;
}

@Override
public String setAppToken(Long userId) {
    String token = DecriptUtil.LoginEncrypt(userId+"");
    stringRedisTemplate.opsForValue().set(CacheKey.APP_TOKEN + userId, token, 30, TimeUnit.DAYS);
    return token;
}

@Override
public boolean checkAppToken(Long userId, String token) {
    String appToken = stringRedisTemplate.opsForValue().get(CacheKey.APP_TOKEN + userId);
    return !EmptyUtils.isEmpty(appToken) && appToken.equals(token);
}


```

## 3、需要拦截的url打入Token

```
@Token
```

```
/**
 * 跳转到优惠券
 * @return
 */
@Token
@GetMapping(value = "itemFilter",produces = "application/json")
@ResponseBody
@ApiOperation(value = "跳转到优惠券",notes = "跳转到优惠券",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE,
        response = CouponItemGood.class)
@ApiImplicitParams({
        @ApiImplicitParam(name = "userInfoId" ,value = "用户的id",dataTypeClass = Long.class,required = true,paramType = "query"),
        @ApiImplicitParam(name = "timeDiff" ,value = "渠道的失效时间设置",dataTypeClass = Integer.class,paramType = "query")
})
public ResponseBean itemFilter(Long userInfoId,Integer timeDiff){
    try {
        return ResponseBean.buildSuccess (haoDaoKuService.itemFilter(WeChatMessageParams.HAO_DAN_KU_GAOYONG_APIKEY, userInfoId, timeDiff));
    }catch (AppException e){
        ExceptionLogUtils.log(e, this.getClass());
        return ResponseBean.buildFailure(e.getCode(),e.getMessage());
    }catch (Exception e){
        ExceptionLogUtils.log(e, this.getClass());
        return ResponseBean.buildFailure(e.getMessage());
    }
}

```


## 4、Token拦截器

```
package com.duodian.youhui.admin.config.token.aop;

import com.duodian.youhui.admin.bean.ResponseBean;
import com.duodian.youhui.admin.config.ContextHolder;
import com.duodian.youhui.admin.config.token.annotation.Token;
import com.duodian.youhui.admin.moudle.user.service.UserInfoService;
import com.duodian.youhui.admin.utils.DecriptUtil;
import com.duodian.youhui.enums.exception.ErrorCodeEnum;
import org.apache.commons.lang.StringUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.annotation.Resource;
import javax.servlet.http.HttpServletRequest;
import java.lang.reflect.Method;

/**
 * @author HealerJean
 */
@Aspect
@Component
public class TokenInterceptor {

    @Resource
    private UserInfoService userInfoService;

    /**
     * 先取method的Token，如果没有再去class的Token，如果有使用method的Token
     * 从request获取token进行解密，获取userId和oid，如果解密失败，返回错误信息
     * 把userId和oid放入当前上下文
     *
     * @param point
     * @return
     * @throws Throwable
     */
    @Around("@annotation(com.duodian.youhui.admin.config.token.annotation.Token) || @within(com.duodian.youhui.admin.config.token.annotation.Token)")
    protected Object invoke(ProceedingJoinPoint point) throws Throwable {
        Token token = null;
        Signature signature = point.getSignature();
        if (signature instanceof MethodSignature) {
            Method method = ((MethodSignature) signature).getMethod();
            token = method.getAnnotation(Token.class);
        }
        if (token == null) {
            token = point.getTarget().getClass().getAnnotation(Token.class);
        }
        if (token == null || !token.check()) {
            return point.proceed();
        }
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        String tokenParam = request.getParameter("token");
        // String tokenParam = request.getHeader("token");
        if (tokenParam == null) {
            return ResponseBean.buildFailure(ErrorCodeEnum.token认证失败)  ;
        }

        try {
            String decrip =  DecriptUtil.LoginDecrypt(tokenParam) ;
            if (decrip == null) {
                return ResponseBean.buildFailure(ErrorCodeEnum.token认证失败)  ;
            }
            Long userId = Long.valueOf( decrip);
            //检验redis中是否还存在，并比较相等
            if (!userInfoService.checkAppToken(userId, tokenParam)) {
                return ResponseBean.buildFailure(ErrorCodeEnum.token认证失败) ;
            }
            ContextHolder.setUserId(userId);
            return point.proceed();
        } catch (Exception e) {
            throw e;
        } finally {
            ContextHolder.clear();
        }
    }

}


```

## 5、Token加解密


```
package com.duodian.youhui.admin.utils;

import sun.misc.BASE64Decoder;
import sun.misc.BASE64Encoder;

import javax.crypto.*;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.io.UnsupportedEncodingException;
import java.security.InvalidKeyException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;

/**
 * @Desc:各种加密解密
 * @Author HealerJean
 * @Date 2018/5/23  下午3:12.
 */

public class DecriptUtil {


    private static final String Loginkey = "jDFBfadsffsdffasdf";
    private static final byte[] Loginiv = "4960432565656562".getBytes();;
    private static final String Loginmode = "AES/CBC/PKCS5Padding";


    public static String LoginEncrypt( String strIn)  {
        try {
            SecretKeySpec skeySpec = getKey(Loginkey);
            Cipher cipher = Cipher.getInstance(Loginmode); //"算法/模式/补码方式"
            IvParameterSpec iv = new IvParameterSpec(Loginiv);////使用CBC模式，需要一个向量iv，可增加加密算法的强度
            cipher.init(Cipher.ENCRYPT_MODE, skeySpec, iv);
            byte[] encrypted = cipher.doFinal(strIn.getBytes());
            return new BASE64Encoder().encode(encrypted);
        } catch (Exception e) {
            return  null;
        }
    }

    public static String LoginDecrypt(String strIn) {
        try {
            SecretKeySpec skeySpec = getKey(Loginkey);
            Cipher cipher = Cipher.getInstance(Loginmode);
            IvParameterSpec iv = new IvParameterSpec(Loginiv);
            cipher.init(Cipher.DECRYPT_MODE, skeySpec, iv);
            byte[] encrypted1 = new BASE64Decoder().decodeBuffer(strIn);

            byte[] original = cipher.doFinal(encrypted1);
            String originalString = new String(original);
            return originalString;
        } catch (Exception e) {
            return null;
        }
    }

    private static SecretKeySpec getKey(String strKey) throws Exception {
        byte[] arrBTmp = strKey.getBytes();
        byte[] arrB = new byte[16]; // 创建一个空的16位字节数组（默认值为0）
        for (int i = 0; i < arrBTmp.length && i < arrB.length; i++) {
            arrB[i] = arrBTmp[i];
        }
        SecretKeySpec skeySpec = new SecretKeySpec(arrB, "AES");
        return skeySpec;
    }





    public static String SHA1(String decript) {
        try {
            MessageDigest digest = MessageDigest
                    .getInstance("SHA-1");
            digest.update(decript.getBytes());
            byte messageDigest[] = digest.digest();
            // Create Hex String
            StringBuffer hexString = new StringBuffer();
            // 字节数组转换为 十六进制 数
            for (int i = 0; i < messageDigest.length; i++) {
                String shaHex = Integer.toHexString(messageDigest[i] & 0xFF);
                if (shaHex.length() < 2) {
                    hexString.append(0);
                }
                hexString.append(shaHex);
            }
            return hexString.toString();

        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        return "";
    }

    public static String SHA(String decript) {
        try {
            MessageDigest digest = MessageDigest
                    .getInstance("SHA");
            digest.update(decript.getBytes());
            byte messageDigest[] = digest.digest();
            // Create Hex String
            StringBuffer hexString = new StringBuffer();
            // 字节数组转换为 十六进制 数
            for (int i = 0; i < messageDigest.length; i++) {
                String shaHex = Integer.toHexString(messageDigest[i] & 0xFF);
                if (shaHex.length() < 2) {
                    hexString.append(0);
                }
                hexString.append(shaHex);
            }
            return hexString.toString();

        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        return "";
    }

    public static String MD5(String input) {
        try {
            // 获得MD5摘要算法的 MessageDigest 对象
            MessageDigest mdInst = MessageDigest.getInstance("MD5");
            // 使用指定的字节更新摘要
            mdInst.update(input.getBytes());
            // 获得密文
            byte[] md = mdInst.digest();
            // 把密文转换成十六进制的字符串形式
            StringBuffer hexString = new StringBuffer();
            // 字节数组转换为 十六进制 数
            for (int i = 0; i < md.length; i++) {
                String shaHex = Integer.toHexString(md[i] & 0xFF);
                if (shaHex.length() < 2) {
                    hexString.append(0);
                }
                hexString.append(shaHex);
            }
            return hexString.toString();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        return "";
    }

    /**
     * 加密
     *
     * @param content
     *            需要加密的内容
     * @param password
     *            加密密码
     * @return
     */
    public static byte[] encryptAES(String content, String password) {
        try {
            KeyGenerator kgen = KeyGenerator.getInstance("AES");
            kgen.init(128, new SecureRandom(password.getBytes()));
            SecretKey secretKey = kgen.generateKey();
            byte[] enCodeFormat = secretKey.getEncoded();
            SecretKeySpec key = new SecretKeySpec(enCodeFormat, "AES");
            Cipher cipher = Cipher.getInstance("AES");// 创建密码器
            byte[] byteContent = content.getBytes("utf-8");
            cipher.init(Cipher.ENCRYPT_MODE, key);// 初始化
            byte[] result = cipher.doFinal(byteContent);
            return result; // 加密
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (NoSuchPaddingException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        } catch (IllegalBlockSizeException e) {
            e.printStackTrace();
        } catch (BadPaddingException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 解密
     *
     * @param content
     *            待解密内容
     * @param password
     *            解密密钥
     * @return
     */
    public static byte[] decryptAES(byte[] content, String password) {
        try {
            KeyGenerator kgen = KeyGenerator.getInstance("AES");
            kgen.init(128, new SecureRandom(password.getBytes()));
            SecretKey secretKey = kgen.generateKey();
            byte[] enCodeFormat = secretKey.getEncoded();
            SecretKeySpec key = new SecretKeySpec(enCodeFormat, "AES");
            Cipher cipher = Cipher.getInstance("AES");// 创建密码器
            cipher.init(Cipher.DECRYPT_MODE, key);// 初始化
            byte[] result = cipher.doFinal(content);
            return result; // 加密
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (NoSuchPaddingException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        } catch (IllegalBlockSizeException e) {
            e.printStackTrace();
        } catch (BadPaddingException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * BASE64解密
     *
     * @param key
     * @return
     * @throws Exception
     */
    public static String decryptBASE64(String key) {

        return "";
    }

    /**
     * BASE64加密
     *
     * @param key
     * @return
     * @throws Exception
     */
    public static String encryptBASE64(String key) {

        return "";
    }


//    public static void main(String[] args) throws Exception {
//
//        String Code = "1";
//        String codE = DecriptUtil.LoginEncrypt(Code);
//        System.out.println("原文：" + Code);
//        System.out.println("密文：" + codE);
//        System.out.println("解密：" + DecriptUtil.LoginDecrypt( codE));
//    }



}

```



<br/><br/><br/>
如果满意，请打赏博主任意金额，感兴趣的在微信转账的时候，添加博主微信哦， 请下方留言吧。可与博主自由讨论哦

|支付包 | 微信|微信公众号|
|:-------:|:-------:|:------:|
|![支付宝](https://raw.githubusercontent.com/HealerJean123/HealerJean123.github.io/master/assets/img/tctip/alpay.jpg) | ![微信](https://raw.githubusercontent.com/HealerJean123/HealerJean123.github.io/master/assets/img/tctip/weixin.jpg)|![微信公众号](https://raw.githubusercontent.com/HealerJean123/HealerJean123.github.io/master/assets/img/my/qrcode_for_gh_a23c07a2da9e_258.jpg)|




<!-- Gitalk 评论 start  -->

<link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
<script src="https://unpkg.com/gitalk@latest/dist/gitalk.min.js"></script> 
<div id="gitalk-container"></div>    
 <script type="text/javascript">
    var gitalk = new Gitalk({
		clientID: `1d164cd85549874d0e3a`,
		clientSecret: `527c3d223d1e6608953e835b547061037d140355`,
		repo: `HealerJean123.github.io`,
		owner: 'HealerJean123',
		admin: ['HealerJean123'],
		id: 'eviknHJocGNuqCdx',
    });
    gitalk.render('gitalk-container');
</script> 

<!-- Gitalk end -->

