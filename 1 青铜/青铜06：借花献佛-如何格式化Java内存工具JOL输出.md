欢迎来到《[王者并发课](https://github.com/ThoughtsBeta/TheKingOfConcurrency)》，本文是该系列文章中的**第6篇**。

在前面的文章《一探究竟-如何从synchronized理解Java对象头中的锁》中，我们介绍并体验了JOL工具。虽然JOL很赞，但它的输出对我们不是很友好，如果不借助工具，我们很难直观理解其中的含义。

下面这段代码是对JOL输出的翻译，建议你收藏。**代码非我所写，文末已经注明出处**。

```java

import org.openjdk.jol.info.ClassLayout;

import java.nio.ByteOrder;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class PrintObjectHeader {
    /**
     * Get binary data
     *
     * @param o
     * @return
     */
    public static String getObjectHeader(Object o) {
        ByteOrder order = ByteOrder.nativeOrder();//Byte order
        String table = ClassLayout.parseInstance(o).toPrintable();
        Pattern p = Pattern.compile("(0|1){8}");
        Matcher matcher = p.matcher(table);
        List<String> header = new ArrayList<String>();
        while (matcher.find()) {
            header.add(matcher.group());
        }
        //Little-endian machines, need to traverse in reverse
        StringBuilder sb = new StringBuilder();
        if (order.equals(ByteOrder.LITTLE_ENDIAN)) {
            Collections.reverse(header);
        }
        for (String s : header) {
            sb.append(s).append(" ");
        }
        return sb.toString().trim();
    }

    /**
     * Parsing object header function for 64bit jvm
     * In 64bit jvm, the object header has two parts: Mark Word and Class Pointer, Mark Word takes 8 bytes, Class Pointer takes 4 bytes
     *
     * @param s Binary string of object header (each 8 bits, separated by a space)
     */
    public static void parseObjectHeader(String s) {
        String[] tmp = s.split(" ");
        System.out.print("Class Pointer: ");
        for (int i = 0; i < 4; ++i) {
            System.out.print(tmp[i] + " ");
        }
        System.out.println("\nMark Word:");
        if (tmp[11].charAt(5) == '0' && tmp[11].substring(6).equals("01")) {//0 01 lock-free state, regardless of GC mark
            //notice: Mark word structure without lock: unused(25bit) + hashcode(31bit) + unused(1bit) + age(4bit) + biased_lock_flag(1bit) + lock_type(2bit)
            // The reason why hashcode only needs 31bit is: hashcode can only be greater than or equal to 0, eliminating the negative range, so you can use 31bit to store
            System.out.print("\thashcode (31bit): ");
            System.out.print(tmp[7].substring(1) + " ");
            for (int i = 8; i < 11; ++i) System.out.print(tmp[i] + " ");
            System.out.println();
        } else if (tmp[11].charAt(5) == '1' && tmp[11].substring(6).equals("01")) {//1 01, which is the case of biased lock
            //notice: The object is in a biased lock, its structure is: ThreadID(54bit) + epoch(2bit) + unused(1bit) + age(4bit) + biased_lock_flag(1bit) + lock_type(2bit)
            // ThreadID here is the thread ID holding the biased lock, epoch: a timestamp of the biased lock, used for optimization of the biased lock
            System.out.print("\tThreadID(54bit): ");
            for (int i = 4; i < 10; ++i) System.out.print(tmp[i] + " ");
            System.out.println(tmp[10].substring(0, 6));
            System.out.println("\tepoch: " + tmp[10].substring(6));
        } else {//In the case of lightweight locks or heavyweight locks, regardless of the GC mark
            //notice: JavaThread*(62bit,include zero padding) + lock_type(2bit)
            // At this point, JavaThread* points to the monitor of the lock record/heavyweight lock in the stack
            System.out.print("\tjavaThread*(62bit,include zero padding): ");
            for (int i = 4; i < 11; ++i) System.out.print(tmp[i] + " ");
            System.out.println(tmp[11].substring(0, 6));
            System.out.println("\tLockFlag (2bit): " + tmp[11].substring(6));
            System.out.println();
            return;
        }
        System.out.println("\tage (4bit): " + tmp[11].substring(1, 5));
        System.out.println("\tbiasedLockFlag (1bit): " + tmp[11].charAt(5));
        System.out.println("\tLockFlag (2bit): " + tmp[11].substring(6));

        System.out.println();
    }

    public static void printObjectHeader(Object o) {
        if (o == null) {
            System.out.println("null object.");
            return;
        }
        parseObjectHeader(getObjectHeader(o));
    }
}

```

## 延伸阅读与参考资料

* https://www.programmersought.com/article/21094532407/
* 掘金专栏：https://juejin.cn/column/6963590682602635294
* github：https://github.com/ThoughtsBeta/TheKingOfConcurrency

## 关于作者

专注高并发领域创作。人气专栏《王者并发课》、小册《[高并发秒杀的设计精要与实现](https://juejin.cn/book/7008372989179723787)》作者，关注公众号【MetaThoughts】，及时获取文章更新和文稿。

---

如果本文对你有帮助，欢迎**点赞**、**关注**、**监督**，我们一起**从青铜到王者**。

