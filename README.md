```java
package com.jackson;

import com.example.demo.Address;
import com.example.demo.User;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.databind.ser.std.StdSerializer;
import java.io.IOException;

/**
 * @author wuzhenhong
 * @date 2024/12/12 9:11
 */
public class T {


    public static class UserSerializer extends StdSerializer<String> {

        public UserSerializer() {
            super(String.class);
        }

        @Override
        public void serialize(String value, JsonGenerator gen, SerializerProvider provider) throws IOException {

            gen.writeStartObject();
            gen.writeStringField(gen.getOutputContext().getParent().getCurrentName() + "脱敏", value + "**");
            gen.writeStringField(gen.getOutputContext().getParent().getCurrentName() + "密码", "xxxx");
            gen.writeEndObject();
        }

    }


    public static void main(String[] args) throws JsonProcessingException {

        User user = new User();
        user.setName("苏城锋");
        user.setAge(29);

        Address address = new Address();
        address.setXxx("北京天安门");
        user.setAddress(address);

        com.fasterxml.jackson.databind.ObjectMapper mapper = new com.fasterxml.jackson.databind.ObjectMapper();
        SimpleModule module = new SimpleModule();
        module.addSerializer(String.class,new UserSerializer());
        mapper.registerModule(module);

        String json = mapper.writeValueAsString(user);
        System.out.println(json);

    }

}

```
