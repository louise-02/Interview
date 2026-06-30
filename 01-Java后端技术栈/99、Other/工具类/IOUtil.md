```java
import lombok.extern.slf4j.Slf4j;

import java.io.Flushable;
import java.io.IOException;

/**
 * IOUtil
 *
 * @author louise
 * @since 2021/7/28 16:20
 */
@Slf4j
public class IOUtil {
    public static void closeAll(AutoCloseable... autoCloseable) {
        for (AutoCloseable closeable : autoCloseable) {
            if (closeable == null) {
                continue;
            }
            if (closeable instanceof Flushable) {
                Flushable flushable = (Flushable) closeable;

                try {
                    flushable.flush();
                } catch (IOException e) {
                    log.error("io flush error", e);
                }
            }

            try {
                closeable.close();
            } catch (Exception e) {
                log.error("io close error", e);
            }

        }
    }
}
```

