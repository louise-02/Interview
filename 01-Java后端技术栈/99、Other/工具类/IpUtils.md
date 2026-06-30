```java
import javax.servlet.http.HttpServletRequest;
import java.net.InetAddress;
import java.net.UnknownHostException;

/**
 * ip工具类
 */
public class IpUtils {
    private static final String UNKNOWN = "unknown";
    private static final String COMMA = ",";

    /**
     * 获取真实IP地址
     */
    public static String getIpAddress(HttpServletRequest request) {
        if (request == null) {
            return UNKNOWN;
        }

        String ip = request.getHeader("x-forwarded-for");
        if (isInvalidIp(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (isInvalidIp(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (isInvalidIp(ip)) {
            ip = request.getHeader("HTTP_CLIENT_IP");
        }
        if (isInvalidIp(ip)) {
            ip = request.getHeader("HTTP_X_FORWARDED_FOR");
        }
        if (isInvalidIp(ip)) {
            ip = request.getRemoteAddr();
            // 处理本地回环地址
            if ("127.0.0.1".equals(ip) || "0:0:0:0:0:0:0:1".equals(ip)) {
                try {
                    // 根据网卡取本机配置的IP
                    ip = InetAddress.getLocalHost().getHostAddress();
                } catch (UnknownHostException e) {
                    // 记录日志或忽略
                }
            }
        }

        // 处理多级代理情况，第一个IP才是真实IP
        if (ip != null && ip.contains(COMMA)) {
            ip = ip.split(COMMA)[0].trim();
        }

        return ip;
    }

    /**
     * 判断IP是否无效
     */
    private static boolean isInvalidIp(String ip) {
        return ip == null || ip.isEmpty() || UNKNOWN.equalsIgnoreCase(ip);
    }
}
```

