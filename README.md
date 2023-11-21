
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

@Component
public class MaskSsnFilter extends AbstractGatewayFilterFactory<Object> {

    private static final String SSN_MASK = "XXX-XX-XXXX"; // Replace with your desired mask
    private static final String SSN_REGEX = "\\b\\d{3}-\\d{2}-\\d{4}\\b";

    public GatewayFilter apply(Object config) {
        return (exchange, chain) -> {
            ServerHttpResponse response = exchange.getResponse();
            DataBuffer originalBuffer = response.bufferFactory().allocateBuffer();
            DataBuffer maskBuffer = response.bufferFactory().allocateBuffer();

            return chain.filter(exchange.mutate().response(originalBuffer).build())
                    .then(Mono.defer(() -> {
                        byte[] originalBytes = new byte[originalBuffer.readableByteCount()];
                        originalBuffer.read(originalBytes);

                        String originalBody = new String(originalBytes, StandardCharsets.UTF_8);

                        String maskedBody = maskSsns(originalBody);

                        byte[] maskBytes = maskedBody.getBytes(StandardCharsets.UTF_8);
                        maskBuffer.write(maskBytes);

                        return response.writeWith(Flux.just(maskBuffer));
                    }));
        };
    }

    private String maskSsns(String input) {
        Pattern pattern = Pattern.compile(SSN_REGEX);
        Matcher matcher = pattern.matcher(input);

        StringBuffer result = new StringBuffer();

        while (matcher.find()) {
            matcher.appendReplacement(result, SSN_MASK);
        }
        matcher.appendTail(result);

        return result.toString();
    }
}
