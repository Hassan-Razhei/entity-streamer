# معمارية Entity Streamer

تعتمد حزمة `entity-streamer` على معمارية مجاري البيانات الثنائية غير المتزامنة (Chunked Binary Streaming Architecture)، مع الالتزام الصارم ببروتوكول طلبات النطاقات الجزئية **RFC 7233**:

```
[ HTTP Client (Browser / Video Player / App) ]
   │
   ├─► Range Request (e.g., "bytes=1048576-2097151")
   ▼
[ Streamer Facade / Response Macro ]
   │
   ├─► PathSanitizer & StreamUrlSigner (التحقق الأمني وحماية المسار)
   ▼
[ StreamableStorageAdapter ]
   │  └── Local / S3 / Custom Flysystem Disk + Fallback Prefixes
   ▼
[ RangeHeaderParser ]
   │  └── تحليل النطاق وحسابه بدقة وفق RFC 7233
   ▼
[ BinaryStreamer Kernel ]
   │  ├── Chunk-by-Chunk Streaming (Buffer: 64 KB)
   │  ├── fseek مباشر لموقع البايت المطلوب
   │  ├── تفريغ الذاكرة المؤقتة (Zero-Ballooning via ob_end_clean)
   │  └── كشف انقطاع العميل (connection_aborted)
   ▼
[ HTTP 206 Partial Content Response ]
```

## المكونات المعمارية الأساسية

1. **RangeHeaderParser**: محلل قياسي لترويسة `Range` يدعم النطاقات المغلقة، المفتوحة، واللاحقة (Suffix)، ويرمي استجابة `416 Range Not Satisfiable` عند عدم مطابقة النطاق.
2. **BinaryStreamer**: محرك الدفق النواة المسؤول عن قراءة الكتل ونقلها مباشرة للمستعرض دون حجز الملف في ذاكرة السيرفر RAM.
3. **StreamableStorageAdapter**: جسر الربط مع محركات تخزين لارافيل (Flysystem v3) مع إمكانية تجربة مسارات بديلة متعددة تلقائياً.
4. **MimeDetector & MediaHeaderManager**: الفحص الذكي للوسائط عبر البايتات السحرية (Magic Bytes) لكافة صيغ الفيديو والصوتيات، وتوليد ترويسات `ETag` و `Last-Modified` مع معالجة رد `304 Not Modified`.
5. **Security Shield**: درع تطهير المسارات (`PathSanitizer`) لمنع هجمات الالتفاف (`..`) والبايتات الصفرية، وتوليد روابط موقتة مشفرة (`StreamUrlSigner`) بخوارزمية HMAC-SHA256 مع حظر سرقة الروابط (Anti-Leech).
6. **HasStreamableMedia Trait**: تكامل أصيل مع نماذج لارافيل Eloquent لبث الوسائط من المودل بسطر كود واحد.

