# 密钥分存Shamir

## 源码

主类：

```java
import java.io.*;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;
import java.util.*;
import java.util.stream.Stream;

/**
 * 密钥分享/分存工具类
 *
 * @see <a href="https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing">Shamir's Secret
 *  *     Sharing</a>
 */
public class Shamir {
    private final static int DEFAULT_SHARES = 5;
    private final static int DEFAULT_MINIMUM = 5;
    private final static String SEPARATOR = "_";
    private final static int READ_BUFFER_SIZE = 512;
    //默认工作目录，用于合并分密钥后的输出
    private final static String DEFAULT_WORKDIR = "/out";
    //通过环境变量传参
    private final static String _WORKDIR = "WORKDIR";
    private final static String _READ_ONLY_MODE = "r";
    private final static String _SPLIT = "split";
    private final static String _JOIN = "join";

    public static void main(String[] args) throws IOException {

        Shamir shamir = new Shamir();
        if (_SPLIT.equals(args[0])) {
            shamir.split(absolutePath(args[1]), DEFAULT_SHARES, DEFAULT_MINIMUM);
        } else if (_JOIN.equals(args[0])) {
            String secretFile = absolutePath(args[1]);
            String[] partFiles = Arrays.copyOfRange(args, 2, args.length);
            partFiles = Stream.of(partFiles).map(Shamir::absolutePath).toArray(String[]::new);
            shamir.join(secretFile,
                    partFiles, DEFAULT_SHARES, DEFAULT_MINIMUM);
        } else {
            throw new UnsupportedOperationException("未知操作");
        }
    }

    private static String absolutePath(String file){
        Objects.requireNonNull(file, "file must not null");
        String path = workdir();
        if (file.startsWith(File.separator)) {
            file = file.substring(File.separator.length());
        }
        if (!path.endsWith(File.separator)) {
            path = path + File.separator;
        }
        return path + file;
    }
    /**
     * 通过环境变量获取工作目录，用于合并密钥输出
     *
     * @return 工作目录
     */
    private static String workdir() {
        String workdir = System.getenv(_WORKDIR);
        if (workdir == null || "".equals(workdir)) {
            workdir = DEFAULT_WORKDIR;
        }
        return workdir;
    }

    /**
     * 合并分密钥
     *
     * @param output 输出路径，如果为null则不输出到文件
     * @param partFiles 分密钥路径
     * @param shares 拆分时设置的个数
     * @param minimum 拆分时设置的最小要求个数
     * @return 原始密钥
     * @throws IOException if
     */
    public byte[] join(String output, String[] partFiles, int shares, int minimum) throws IOException {
        if (partFiles == null) {
            throw new IllegalArgumentException("partFiles is null");
        }

        int index = 0;
        Map<Integer, byte[]> parts = new HashMap<>();
        for (String partFile : partFiles) {
            byte[] part = readBytes(partFile);
            parts.put(++index, part);
        }

        final byte[] secret = new Scheme(new SecureRandom(), shares, minimum)
                .join(parts);

        if (output != null) {
            writeBytes(output, secret);
        }
        return secret;
    }

    /**
     * 分割密钥
     *
     * @param secretFile 密钥文件路径
     * @param shares 分割个数
     * @param minimum 合并最小要求个数
     * @return 分割的密钥
     * @throws IOException if
     */
    public Map<Integer, byte[]> split(String secretFile, int shares, int minimum) throws IOException {

        final byte[] secret = readBytes(secretFile);
        if (secret == null)
            return Collections.emptyMap();
        Scheme scheme = new Scheme(new SecureRandom(), shares, minimum);

        Map<Integer, byte[]> parts = scheme.split(secret);

        for (Map.Entry<Integer, byte[]> entry : parts.entrySet()) {
            String filename = secretFile + SEPARATOR + entry.getKey();
            writeBytes(filename, entry.getValue());
        }
        return parts;
    }

    /**
     * 将内存中字节数组写入文件
     * @param filename 文件名
     * @param bytes 字节数组
     */
    private void writeBytes(String filename, byte[] bytes) throws IOException{
        File file = new File(filename);
        if (!file.exists() && !file.createNewFile()) {
            throw new RuntimeException("create new file fail:" + filename);
        }
        try (FileOutputStream fileOutputStream = new FileOutputStream(file);
             FileChannel outputChannel = fileOutputStream.getChannel()) {

            ByteBuffer buffer = ByteBuffer.allocate(bytes.length);
            buffer.put(bytes);
            buffer.flip();
            outputChannel.write(buffer);
        }
    }

    /**
     * 读取文件到内存
     * @param file 文件路径字符串
     * @return 字节数组
     * @throws IOException if
     */
    private byte[] readBytes(String file) throws IOException {
        byte[] bytes = null;

        try (RandomAccessFile r = new RandomAccessFile(file, _READ_ONLY_MODE);
             FileChannel inChannel = r.getChannel()) {

            ByteBuffer buffer = ByteBuffer.allocate(READ_BUFFER_SIZE);
            int length = -1;

            while ((length = inChannel.read(buffer)) != -1) {
                buffer.flip();
                int dataLength = buffer.limit() - buffer.position();
                bytes = mergeArray(bytes, Arrays.copyOf(buffer.array(), dataLength));
                buffer.clear();
            }
        }
        return bytes;
    }

    /**
     * 合并数组
     *
     * @param array1 数组1
     * @param array2 数组2
     * @return 新数组，包含数组1和数组2
     */
    private static byte[] mergeArray(final byte[] array1, final byte[] array2) {
        if (array1 == null) {
            return clone(array2);
        } else if (array2 == null) {
            return clone(array1);
        }
        final byte[] joinedArray = new byte[array1.length + array2.length];
        System.arraycopy(array1, 0, joinedArray, 0, array1.length);
        System.arraycopy(array2, 0, joinedArray, array1.length, array2.length);
        return joinedArray;
    }

    /**
     * 复制数组
     *
     * @param array 目标数组
     * @return 新数组
     */
    private static byte[] clone(final byte[] array) {
        if (array == null) {
            return null;
        }
        return array.clone();
    }

}

```

算法类：

```java
/*
 * Copyright © 2017 Coda Hale (coda.hale@gmail.com)
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

import java.security.SecureRandom;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
import java.util.StringJoiner;

/**
 * An implementation of Shamir's Secret Sharing over {@code GF(256)} to securely split secrets into
 * {@code N} parts, of which any {@code K} can be joined to recover the original secret.
 *
 * <p>{@link Scheme} uses the same GF(256) field polynomial as the Advanced Encryption Standard
 * (AES): {@code 0x11b}, or {@code x}<sup>8</sup> + {@code x}<sup>4</sup> + {@code x}<sup>3</sup> +
 * {@code x} + 1.
 *
 * @see <a href="https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing">Shamir's Secret
 *     Sharing</a>
 * @see <a href="http://www.cs.utsa.edu/~wagner/laws/FFM.html">The Finite Field {@code GF(256)}</a>
 */
public class Scheme {

  private final SecureRandom random;
  private final int n;
  private final int k;

  /**
   * Creates a new {@link Scheme} instance.
   *
   * @param random a {@link SecureRandom} instance
   * @param n the number of parts to produce (must be {@code >1})
   * @param k the threshold of joinable parts (must be {@code <= n})
   */
  public Scheme(SecureRandom random, int n, int k) {
    this.random = random;
    checkArgument(k > 1, "K must be > 1");
    checkArgument(n >= k, "N must be >= K");
    checkArgument(n <= 255, "N must be <= 255");
    this.n = n;
    this.k = k;
  }

  /**
   * Splits the given secret into {@code n} parts, of which any {@code k} or more can be combined to
   * recover the original secret.
   *
   * @param secret the secret to split
   * @return a map of {@code n} part IDs and their values
   */
  public Map<Integer, byte[]> split(byte[] secret) {
    // generate part values
    final byte[][] values = new byte[n][secret.length];
    for (int i = 0; i < secret.length; i++) {
      // for each byte, generate a random polynomial, p
      final byte[] p = GF256.generate(random, k - 1, secret[i]);
      for (int x = 1; x <= n; x++) {
        // each part's byte is p(partId)
        values[x - 1][i] = GF256.eval(p, (byte) x);
      }
    }

    // return as a set of objects
    final Map<Integer, byte[]> parts = new HashMap<>(n());
    for (int i = 0; i < values.length; i++) {
      parts.put(i + 1, values[i]);
    }
    return Collections.unmodifiableMap(parts);
  }

  /**
   * Joins the given parts to recover the original secret.
   *
   * <p><b>N.B.:</b> There is no way to determine whether or not the returned value is actually the
   * original secret. If the parts are incorrect, or are under the threshold value used to split the
   * secret, a random value will be returned.
   *
   * @param parts a map of part IDs to part values
   * @return the original secret
   * @throws IllegalArgumentException if {@code parts} is empty or contains values of varying
   *     lengths
   */
  public byte[] join(Map<Integer, byte[]> parts) {
    checkArgument(parts.size() > 0, "No parts provided");
    final int[] lengths = parts.values().stream().mapToInt(v -> v.length).distinct().toArray();
    checkArgument(lengths.length == 1, "Varying lengths of part values");
    final byte[] secret = new byte[lengths[0]];
    for (int i = 0; i < secret.length; i++) {
      final byte[][] points = new byte[parts.size()][2];
      int j = 0;
      for (Map.Entry<Integer, byte[]> part : parts.entrySet()) {
        points[j][0] = part.getKey().byteValue();
        points[j][1] = part.getValue()[i];
        j++;
      }
      secret[i] = GF256.interpolate(points);
    }
    return secret;
  }

  /**
   * The number of parts the scheme will generate when splitting a secret.
   *
   * @return {@code N}
   */
  public int n() {
    return n;
  }

  /**
   * The number of parts the scheme will require to re-create a secret.
   *
   * @return {@code K}
   */
  public int k() {
    return k;
  }

  @Override
  public boolean equals(Object o) {
    if (this == o) {
      return true;
    }
    if (o == null || getClass() != o.getClass()) {
      return false;
    }
    Scheme scheme = (Scheme) o;
    return n == scheme.n && k == scheme.k && Objects.equals(random, scheme.random);
  }

  @Override
  public int hashCode() {
    return Objects.hash(random, n, k);
  }

  @Override
  public String toString() {
    return new StringJoiner(", ", Scheme.class.getSimpleName() + "[", "]")
        .add("random=" + random)
        .add("n=" + n)
        .add("k=" + k)
        .toString();
  }

  private static void checkArgument(boolean condition, String message) {
    if (!condition) {
      throw new IllegalArgumentException(message);
    }
  }
}


/*
 * Copyright © 2017 Coda Hale (coda.hale@gmail.com)
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

import java.security.SecureRandom;

import static java.lang.Byte.toUnsignedInt;

/**
 * An implementation of polynomials over {@code GF(256)}. Uses the same field polynomial ({@code
 * 0x11b}) and generator ({@code 0x03}) as AES. Internally, uses lookup tables for performance.
 *
 * @see <a href="https://research.swtch.com/field">Finite Field Arithmetic and Reed-Solomon
 *     Coding</a>
 */
class GF256 {
  private GF256() {
    // a singleton
  }

  private static final byte[] LOG = {
    (byte) 0xff, (byte) 0x00, (byte) 0x19, (byte) 0x01, (byte) 0x32, (byte) 0x02, (byte) 0x1a,
    (byte) 0xc6, (byte) 0x4b, (byte) 0xc7, (byte) 0x1b, (byte) 0x68, (byte) 0x33, (byte) 0xee,
    (byte) 0xdf, (byte) 0x03, (byte) 0x64, (byte) 0x04, (byte) 0xe0, (byte) 0x0e, (byte) 0x34,
    (byte) 0x8d, (byte) 0x81, (byte) 0xef, (byte) 0x4c, (byte) 0x71, (byte) 0x08, (byte) 0xc8,
    (byte) 0xf8, (byte) 0x69, (byte) 0x1c, (byte) 0xc1, (byte) 0x7d, (byte) 0xc2, (byte) 0x1d,
    (byte) 0xb5, (byte) 0xf9, (byte) 0xb9, (byte) 0x27, (byte) 0x6a, (byte) 0x4d, (byte) 0xe4,
    (byte) 0xa6, (byte) 0x72, (byte) 0x9a, (byte) 0xc9, (byte) 0x09, (byte) 0x78, (byte) 0x65,
    (byte) 0x2f, (byte) 0x8a, (byte) 0x05, (byte) 0x21, (byte) 0x0f, (byte) 0xe1, (byte) 0x24,
    (byte) 0x12, (byte) 0xf0, (byte) 0x82, (byte) 0x45, (byte) 0x35, (byte) 0x93, (byte) 0xda,
    (byte) 0x8e, (byte) 0x96, (byte) 0x8f, (byte) 0xdb, (byte) 0xbd, (byte) 0x36, (byte) 0xd0,
    (byte) 0xce, (byte) 0x94, (byte) 0x13, (byte) 0x5c, (byte) 0xd2, (byte) 0xf1, (byte) 0x40,
    (byte) 0x46, (byte) 0x83, (byte) 0x38, (byte) 0x66, (byte) 0xdd, (byte) 0xfd, (byte) 0x30,
    (byte) 0xbf, (byte) 0x06, (byte) 0x8b, (byte) 0x62, (byte) 0xb3, (byte) 0x25, (byte) 0xe2,
    (byte) 0x98, (byte) 0x22, (byte) 0x88, (byte) 0x91, (byte) 0x10, (byte) 0x7e, (byte) 0x6e,
    (byte) 0x48, (byte) 0xc3, (byte) 0xa3, (byte) 0xb6, (byte) 0x1e, (byte) 0x42, (byte) 0x3a,
    (byte) 0x6b, (byte) 0x28, (byte) 0x54, (byte) 0xfa, (byte) 0x85, (byte) 0x3d, (byte) 0xba,
    (byte) 0x2b, (byte) 0x79, (byte) 0x0a, (byte) 0x15, (byte) 0x9b, (byte) 0x9f, (byte) 0x5e,
    (byte) 0xca, (byte) 0x4e, (byte) 0xd4, (byte) 0xac, (byte) 0xe5, (byte) 0xf3, (byte) 0x73,
    (byte) 0xa7, (byte) 0x57, (byte) 0xaf, (byte) 0x58, (byte) 0xa8, (byte) 0x50, (byte) 0xf4,
    (byte) 0xea, (byte) 0xd6, (byte) 0x74, (byte) 0x4f, (byte) 0xae, (byte) 0xe9, (byte) 0xd5,
    (byte) 0xe7, (byte) 0xe6, (byte) 0xad, (byte) 0xe8, (byte) 0x2c, (byte) 0xd7, (byte) 0x75,
    (byte) 0x7a, (byte) 0xeb, (byte) 0x16, (byte) 0x0b, (byte) 0xf5, (byte) 0x59, (byte) 0xcb,
    (byte) 0x5f, (byte) 0xb0, (byte) 0x9c, (byte) 0xa9, (byte) 0x51, (byte) 0xa0, (byte) 0x7f,
    (byte) 0x0c, (byte) 0xf6, (byte) 0x6f, (byte) 0x17, (byte) 0xc4, (byte) 0x49, (byte) 0xec,
    (byte) 0xd8, (byte) 0x43, (byte) 0x1f, (byte) 0x2d, (byte) 0xa4, (byte) 0x76, (byte) 0x7b,
    (byte) 0xb7, (byte) 0xcc, (byte) 0xbb, (byte) 0x3e, (byte) 0x5a, (byte) 0xfb, (byte) 0x60,
    (byte) 0xb1, (byte) 0x86, (byte) 0x3b, (byte) 0x52, (byte) 0xa1, (byte) 0x6c, (byte) 0xaa,
    (byte) 0x55, (byte) 0x29, (byte) 0x9d, (byte) 0x97, (byte) 0xb2, (byte) 0x87, (byte) 0x90,
    (byte) 0x61, (byte) 0xbe, (byte) 0xdc, (byte) 0xfc, (byte) 0xbc, (byte) 0x95, (byte) 0xcf,
    (byte) 0xcd, (byte) 0x37, (byte) 0x3f, (byte) 0x5b, (byte) 0xd1, (byte) 0x53, (byte) 0x39,
    (byte) 0x84, (byte) 0x3c, (byte) 0x41, (byte) 0xa2, (byte) 0x6d, (byte) 0x47, (byte) 0x14,
    (byte) 0x2a, (byte) 0x9e, (byte) 0x5d, (byte) 0x56, (byte) 0xf2, (byte) 0xd3, (byte) 0xab,
    (byte) 0x44, (byte) 0x11, (byte) 0x92, (byte) 0xd9, (byte) 0x23, (byte) 0x20, (byte) 0x2e,
    (byte) 0x89, (byte) 0xb4, (byte) 0x7c, (byte) 0xb8, (byte) 0x26, (byte) 0x77, (byte) 0x99,
    (byte) 0xe3, (byte) 0xa5, (byte) 0x67, (byte) 0x4a, (byte) 0xed, (byte) 0xde, (byte) 0xc5,
    (byte) 0x31, (byte) 0xfe, (byte) 0x18, (byte) 0x0d, (byte) 0x63, (byte) 0x8c, (byte) 0x80,
    (byte) 0xc0, (byte) 0xf7, (byte) 0x70, (byte) 0x07,
  };
  private static final byte[] EXP = {
    (byte) 0x01, (byte) 0x03, (byte) 0x05, (byte) 0x0f, (byte) 0x11, (byte) 0x33, (byte) 0x55,
    (byte) 0xff, (byte) 0x1a, (byte) 0x2e, (byte) 0x72, (byte) 0x96, (byte) 0xa1, (byte) 0xf8,
    (byte) 0x13, (byte) 0x35, (byte) 0x5f, (byte) 0xe1, (byte) 0x38, (byte) 0x48, (byte) 0xd8,
    (byte) 0x73, (byte) 0x95, (byte) 0xa4, (byte) 0xf7, (byte) 0x02, (byte) 0x06, (byte) 0x0a,
    (byte) 0x1e, (byte) 0x22, (byte) 0x66, (byte) 0xaa, (byte) 0xe5, (byte) 0x34, (byte) 0x5c,
    (byte) 0xe4, (byte) 0x37, (byte) 0x59, (byte) 0xeb, (byte) 0x26, (byte) 0x6a, (byte) 0xbe,
    (byte) 0xd9, (byte) 0x70, (byte) 0x90, (byte) 0xab, (byte) 0xe6, (byte) 0x31, (byte) 0x53,
    (byte) 0xf5, (byte) 0x04, (byte) 0x0c, (byte) 0x14, (byte) 0x3c, (byte) 0x44, (byte) 0xcc,
    (byte) 0x4f, (byte) 0xd1, (byte) 0x68, (byte) 0xb8, (byte) 0xd3, (byte) 0x6e, (byte) 0xb2,
    (byte) 0xcd, (byte) 0x4c, (byte) 0xd4, (byte) 0x67, (byte) 0xa9, (byte) 0xe0, (byte) 0x3b,
    (byte) 0x4d, (byte) 0xd7, (byte) 0x62, (byte) 0xa6, (byte) 0xf1, (byte) 0x08, (byte) 0x18,
    (byte) 0x28, (byte) 0x78, (byte) 0x88, (byte) 0x83, (byte) 0x9e, (byte) 0xb9, (byte) 0xd0,
    (byte) 0x6b, (byte) 0xbd, (byte) 0xdc, (byte) 0x7f, (byte) 0x81, (byte) 0x98, (byte) 0xb3,
    (byte) 0xce, (byte) 0x49, (byte) 0xdb, (byte) 0x76, (byte) 0x9a, (byte) 0xb5, (byte) 0xc4,
    (byte) 0x57, (byte) 0xf9, (byte) 0x10, (byte) 0x30, (byte) 0x50, (byte) 0xf0, (byte) 0x0b,
    (byte) 0x1d, (byte) 0x27, (byte) 0x69, (byte) 0xbb, (byte) 0xd6, (byte) 0x61, (byte) 0xa3,
    (byte) 0xfe, (byte) 0x19, (byte) 0x2b, (byte) 0x7d, (byte) 0x87, (byte) 0x92, (byte) 0xad,
    (byte) 0xec, (byte) 0x2f, (byte) 0x71, (byte) 0x93, (byte) 0xae, (byte) 0xe9, (byte) 0x20,
    (byte) 0x60, (byte) 0xa0, (byte) 0xfb, (byte) 0x16, (byte) 0x3a, (byte) 0x4e, (byte) 0xd2,
    (byte) 0x6d, (byte) 0xb7, (byte) 0xc2, (byte) 0x5d, (byte) 0xe7, (byte) 0x32, (byte) 0x56,
    (byte) 0xfa, (byte) 0x15, (byte) 0x3f, (byte) 0x41, (byte) 0xc3, (byte) 0x5e, (byte) 0xe2,
    (byte) 0x3d, (byte) 0x47, (byte) 0xc9, (byte) 0x40, (byte) 0xc0, (byte) 0x5b, (byte) 0xed,
    (byte) 0x2c, (byte) 0x74, (byte) 0x9c, (byte) 0xbf, (byte) 0xda, (byte) 0x75, (byte) 0x9f,
    (byte) 0xba, (byte) 0xd5, (byte) 0x64, (byte) 0xac, (byte) 0xef, (byte) 0x2a, (byte) 0x7e,
    (byte) 0x82, (byte) 0x9d, (byte) 0xbc, (byte) 0xdf, (byte) 0x7a, (byte) 0x8e, (byte) 0x89,
    (byte) 0x80, (byte) 0x9b, (byte) 0xb6, (byte) 0xc1, (byte) 0x58, (byte) 0xe8, (byte) 0x23,
    (byte) 0x65, (byte) 0xaf, (byte) 0xea, (byte) 0x25, (byte) 0x6f, (byte) 0xb1, (byte) 0xc8,
    (byte) 0x43, (byte) 0xc5, (byte) 0x54, (byte) 0xfc, (byte) 0x1f, (byte) 0x21, (byte) 0x63,
    (byte) 0xa5, (byte) 0xf4, (byte) 0x07, (byte) 0x09, (byte) 0x1b, (byte) 0x2d, (byte) 0x77,
    (byte) 0x99, (byte) 0xb0, (byte) 0xcb, (byte) 0x46, (byte) 0xca, (byte) 0x45, (byte) 0xcf,
    (byte) 0x4a, (byte) 0xde, (byte) 0x79, (byte) 0x8b, (byte) 0x86, (byte) 0x91, (byte) 0xa8,
    (byte) 0xe3, (byte) 0x3e, (byte) 0x42, (byte) 0xc6, (byte) 0x51, (byte) 0xf3, (byte) 0x0e,
    (byte) 0x12, (byte) 0x36, (byte) 0x5a, (byte) 0xee, (byte) 0x29, (byte) 0x7b, (byte) 0x8d,
    (byte) 0x8c, (byte) 0x8f, (byte) 0x8a, (byte) 0x85, (byte) 0x94, (byte) 0xa7, (byte) 0xf2,
    (byte) 0x0d, (byte) 0x17, (byte) 0x39, (byte) 0x4b, (byte) 0xdd, (byte) 0x7c, (byte) 0x84,
    (byte) 0x97, (byte) 0xa2, (byte) 0xfd, (byte) 0x1c, (byte) 0x24, (byte) 0x6c, (byte) 0xb4,
    (byte) 0xc7, (byte) 0x52, (byte) 0xf6, (byte) 0x01, (byte) 0x03, (byte) 0x05, (byte) 0x0f,
    (byte) 0x11, (byte) 0x33, (byte) 0x55, (byte) 0xff, (byte) 0x1a, (byte) 0x2e, (byte) 0x72,
    (byte) 0x96, (byte) 0xa1, (byte) 0xf8, (byte) 0x13, (byte) 0x35, (byte) 0x5f, (byte) 0xe1,
    (byte) 0x38, (byte) 0x48, (byte) 0xd8, (byte) 0x73, (byte) 0x95, (byte) 0xa4, (byte) 0xf7,
    (byte) 0x02, (byte) 0x06, (byte) 0x0a, (byte) 0x1e, (byte) 0x22, (byte) 0x66, (byte) 0xaa,
    (byte) 0xe5, (byte) 0x34, (byte) 0x5c, (byte) 0xe4, (byte) 0x37, (byte) 0x59, (byte) 0xeb,
    (byte) 0x26, (byte) 0x6a, (byte) 0xbe, (byte) 0xd9, (byte) 0x70, (byte) 0x90, (byte) 0xab,
    (byte) 0xe6, (byte) 0x31, (byte) 0x53, (byte) 0xf5, (byte) 0x04, (byte) 0x0c, (byte) 0x14,
    (byte) 0x3c, (byte) 0x44, (byte) 0xcc, (byte) 0x4f, (byte) 0xd1, (byte) 0x68, (byte) 0xb8,
    (byte) 0xd3, (byte) 0x6e, (byte) 0xb2, (byte) 0xcd, (byte) 0x4c, (byte) 0xd4, (byte) 0x67,
    (byte) 0xa9, (byte) 0xe0, (byte) 0x3b, (byte) 0x4d, (byte) 0xd7, (byte) 0x62, (byte) 0xa6,
    (byte) 0xf1, (byte) 0x08, (byte) 0x18, (byte) 0x28, (byte) 0x78, (byte) 0x88, (byte) 0x83,
    (byte) 0x9e, (byte) 0xb9, (byte) 0xd0, (byte) 0x6b, (byte) 0xbd, (byte) 0xdc, (byte) 0x7f,
    (byte) 0x81, (byte) 0x98, (byte) 0xb3, (byte) 0xce, (byte) 0x49, (byte) 0xdb, (byte) 0x76,
    (byte) 0x9a, (byte) 0xb5, (byte) 0xc4, (byte) 0x57, (byte) 0xf9, (byte) 0x10, (byte) 0x30,
    (byte) 0x50, (byte) 0xf0, (byte) 0x0b, (byte) 0x1d, (byte) 0x27, (byte) 0x69, (byte) 0xbb,
    (byte) 0xd6, (byte) 0x61, (byte) 0xa3, (byte) 0xfe, (byte) 0x19, (byte) 0x2b, (byte) 0x7d,
    (byte) 0x87, (byte) 0x92, (byte) 0xad, (byte) 0xec, (byte) 0x2f, (byte) 0x71, (byte) 0x93,
    (byte) 0xae, (byte) 0xe9, (byte) 0x20, (byte) 0x60, (byte) 0xa0, (byte) 0xfb, (byte) 0x16,
    (byte) 0x3a, (byte) 0x4e, (byte) 0xd2, (byte) 0x6d, (byte) 0xb7, (byte) 0xc2, (byte) 0x5d,
    (byte) 0xe7, (byte) 0x32, (byte) 0x56, (byte) 0xfa, (byte) 0x15, (byte) 0x3f, (byte) 0x41,
    (byte) 0xc3, (byte) 0x5e, (byte) 0xe2, (byte) 0x3d, (byte) 0x47, (byte) 0xc9, (byte) 0x40,
    (byte) 0xc0, (byte) 0x5b, (byte) 0xed, (byte) 0x2c, (byte) 0x74, (byte) 0x9c, (byte) 0xbf,
    (byte) 0xda, (byte) 0x75, (byte) 0x9f, (byte) 0xba, (byte) 0xd5, (byte) 0x64, (byte) 0xac,
    (byte) 0xef, (byte) 0x2a, (byte) 0x7e, (byte) 0x82, (byte) 0x9d, (byte) 0xbc, (byte) 0xdf,
    (byte) 0x7a, (byte) 0x8e, (byte) 0x89, (byte) 0x80, (byte) 0x9b, (byte) 0xb6, (byte) 0xc1,
    (byte) 0x58, (byte) 0xe8, (byte) 0x23, (byte) 0x65, (byte) 0xaf, (byte) 0xea, (byte) 0x25,
    (byte) 0x6f, (byte) 0xb1, (byte) 0xc8, (byte) 0x43, (byte) 0xc5, (byte) 0x54, (byte) 0xfc,
    (byte) 0x1f, (byte) 0x21, (byte) 0x63, (byte) 0xa5, (byte) 0xf4, (byte) 0x07, (byte) 0x09,
    (byte) 0x1b, (byte) 0x2d, (byte) 0x77, (byte) 0x99, (byte) 0xb0, (byte) 0xcb, (byte) 0x46,
    (byte) 0xca, (byte) 0x45, (byte) 0xcf, (byte) 0x4a, (byte) 0xde, (byte) 0x79, (byte) 0x8b,
    (byte) 0x86, (byte) 0x91, (byte) 0xa8, (byte) 0xe3, (byte) 0x3e, (byte) 0x42, (byte) 0xc6,
    (byte) 0x51, (byte) 0xf3, (byte) 0x0e, (byte) 0x12, (byte) 0x36, (byte) 0x5a, (byte) 0xee,
    (byte) 0x29, (byte) 0x7b, (byte) 0x8d, (byte) 0x8c, (byte) 0x8f, (byte) 0x8a, (byte) 0x85,
    (byte) 0x94, (byte) 0xa7, (byte) 0xf2, (byte) 0x0d, (byte) 0x17, (byte) 0x39, (byte) 0x4b,
    (byte) 0xdd, (byte) 0x7c, (byte) 0x84, (byte) 0x97, (byte) 0xa2, (byte) 0xfd, (byte) 0x1c,
    (byte) 0x24, (byte) 0x6c, (byte) 0xb4, (byte) 0xc7, (byte) 0x52, (byte) 0xf6, (byte) 0x00,
    (byte) 0x00
  };

  static byte add(byte a, byte b) {
    return (byte) (a ^ b);
  }

  static byte sub(byte a, byte b) {
    return add(a, b);
  }

  static byte mul(byte a, byte b) {
    if (a == 0 || b == 0) {
      return 0;
    }
    return EXP[toUnsignedInt(LOG[toUnsignedInt(a)]) + toUnsignedInt(LOG[toUnsignedInt(b)])];
  }

  static byte div(byte a, byte b) {
    // multiply by the inverse of b
    return mul(a, EXP[255 - toUnsignedInt(LOG[toUnsignedInt(b)])]);
  }

  static byte eval(byte[] p, byte x) {
    // Horner's method
    byte result = 0;
    for (int i = p.length - 1; i >= 0; i--) {
      result = add(mul(result, x), p[i]);
    }
    return result;
  }

  static int degree(byte[] p) {
    for (int i = p.length - 1; i >= 1; i--) {
      if (p[i] != 0) {
        return i;
      }
    }
    return 0;
  }

  static byte[] generate(SecureRandom random, int degree, byte x) {
    final byte[] p = new byte[degree + 1];

    // generate random polynomials until we find one of the given degree
    do {
      random.nextBytes(p);
    } while (degree(p) != degree);

    // set y intercept
    p[0] = x;

    return p;
  }

  static byte interpolate(byte[][] points) {
    // calculate f(0) of the given points using Lagrangian interpolation
    final byte x = 0;
    byte y = 0;
    for (int i = 0; i < points.length; i++) {
      final byte aX = points[i][0];
      final byte aY = points[i][1];
      byte li = 1;
      for (int j = 0; j < points.length; j++) {
        final byte bX = points[j][0];
        if (i != j) {
          li = mul(li, div(sub(x, bX), sub(aX, bX)));
        }
      }
      y = add(y, mul(li, aY));
    }
    return y;
  }
}

```

## Dockerfile

```java
FROM adoptopenjdk/openjdk8-openj9:alpine-slim
MAINTAINER jerry.wong@lednets.com

ENV WORKDIR /out
ENV CLASSPATH .:/:$JAVA_HOME/jre/lib/ext:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

RUN mkdir $WORKDIR
COPY ./*.class /

WORKDIR $WORKDIR
ENTRYPOINT ["java","Shamir"]

```

## 使用

```shell
docker run --rm -v $PWD:/out colorlightwzg/shamir:v1.0 join merge.secret \
clt.secret_1 clt.secret_2 clt.secret_3 clt.secret_4 clt.secret_5


docker run --rm -v $PWD:/out colorlightwzg/shamir:v1.0 split /out/clt.secret

```

