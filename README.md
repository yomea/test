```java
package com.example.demo;

import cn.hutool.json.JSONUtil;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.databind.ser.std.StdSerializer;
import java.io.IOException;
import java.io.OutputStream;
import java.lang.reflect.Field;
import java.util.Arrays;
import java.util.Objects;
import org.springframework.core.MethodParameter;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

/**
 * @author wuzhenhong
 * @date 2024/12/12 8:22
 */
@ControllerAdvice(basePackageClasses = TestController.class)
public class ResponseBodyAdviceImpl implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
        Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request,
        ServerHttpResponse response) {

        // 这个可以自己封装下成util
        com.fasterxml.jackson.databind.ObjectMapper mapper = new com.fasterxml.jackson.databind.ObjectMapper();
        SimpleModule module = new SimpleModule();
        module.addSerializer(returnType.getParameterType(), new CustomeSerializer(returnType));
        mapper.registerModule(module);

        String json = null;
        try {
            json = mapper.writeValueAsString(body);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }

        response.getHeaders().add("Content-Type",selectedContentType.toString());
        try (OutputStream outputStream = response.getBody()){
            outputStream.write(json.getBytes());
            response.flush();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        return null;
    }


    public static class CustomeSerializer extends StdSerializer<Object> {
        private MethodParameter returnType;
        public CustomeSerializer(MethodParameter returnType) {
            super(Object.class);
            this.returnType = returnType;
        }

        @Override
        public void serialize(Object value, JsonGenerator gen, SerializerProvider provider) throws IOException {

            String name = gen.getOutputContext().getCurrentName();
            try {
                Field field = gen.getOutputContext().getCurrentValue().getClass().getDeclaredField("name");
                在字段上的注解 aaa = field.getAnnotation(在字段上的注解.class);
                if(Objects.nonNull(aaa)) {
                    Handler handler = aaa.newInstance();
                    String str = trimQuotes(value);
                    String mask = handler.mask(str);
                    gen.writeStartObject();
                    gen.writeStringField(name + "脱敏", str);
                    gen.writeStringField(name + "密码", mask);
                    gen.writeEndObject();
                    return;
                }
            } catch (NoSuchFieldException e) {
                throw new RuntimeException(e);
            }
            
            Sensitive sensitiveAnnotation = returnType.getAnnotatedElement().getAnnotation(sensitive.class);
            if(sensitiveAnnotation == null) {
                gen.writeObject(value);
                return;
            }
            FieldHandlerPair[] fieldHandlerPairs = sensitiveAnnotation.value();
            if (fieldHandlerPairs == null || fieldHandlerPairs.length == 0) {
                gen.writeObject(value);
                return;
            }

            for (FieldHandlerPair fieldHandlerPair : fieldHandlerPairs) {
                String fieldName = fieldHandlerPair.fieldName();
                String name = gen.getOutputContext().getCurrentName();
                if(gen.getOutputContext().getCurrentName().equals(fieldName)) {
                    Handler handler = hc.newInstance();
                    String str = trimQuotes(value);
                    String mask = handler.mask(str);
                    gen.writeStartObject();
                    gen.writeStringField(name + "脱敏", str);
                    gen.writeStringField(name + "密码", mask);
                    gen.writeEndObject();
                }
            }
        }
    }
}


```
