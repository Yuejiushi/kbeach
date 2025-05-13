# MMDB版本差异

 GeoLite2 数据库不同版本的区别，列举在表格中：

| **数据库**                | **Edition ID**       | **格式**              | **描述**                                                     | **最后更新时间** |
| ------------------------- | -------------------- | --------------------- | ------------------------------------------------------------ | ---------------- |
| **GeoLite2 ASN**          | GeoLite2-ASN         | GeoIP2 Binary (.mmdb) | 用于提供 IP 地址的自治系统编号 (ASN)，主要用于网络路由和 ISP 信息查询。 | 2024-08-15       |
| **GeoLite2 ASN: CSV**     | GeoLite2-ASN-CSV     | GeoIP2 CSV            | 与 GeoLite2 ASN 数据库类似，但以 CSV 格式提供，便于非 API 的使用。 | 2024-08-15       |
| **GeoLite2 City**         | GeoLite2-City        | GeoIP2 Binary (.mmdb) | 提供城市级别的地理定位数据，包括城市、州、省、邮政编码、经纬度等信息，适用于更精细的地理查询。 | 2024-08-13       |
| **GeoLite2 City: CSV**    | GeoLite2-City-CSV    | GeoIP2 CSV            | 与 GeoLite2 City 数据库类似，但以 CSV 格式提供，便于数据导入和处理。 | 2024-08-13       |
| **GeoLite2 Country**      | GeoLite2-Country     | GeoIP2 Binary (.mmdb) | 提供国家级别的地理定位数据，主要用于确定 IP 地址所属的国家或地区。 | 2024-08-13       |
| **GeoLite2 Country: CSV** | GeoLite2-Country-CSV | GeoIP2 CSV            | 与 GeoLite2 Country 数据库类似，但以 CSV 格式提供，用于更简单的数据分析和集成。 | 2024-08-13       |

### 说明：

- **GeoIP2 Binary (.mmdb)** 格式：用于高效查询的二进制数据库文件，通常与 MaxMind 的 API 一起使用。
- **GeoIP2 CSV** 格式：以逗号分隔值 (CSV) 格式提供，便于使用常见数据处理工具或导入数据库进行处理。

每个版本主要在数据粒度和使用场景上有所不同，用户可以根据需求选择合适的数据库。