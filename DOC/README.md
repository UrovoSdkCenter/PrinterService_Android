# UROVO 打印服务 - Android标准打印服务

PrinterService 是一个基于 Android 标准打印框架的打印服务应用，支持内置打印机和蓝牙打印机，可为网页、应用等多种场景提供打印能力。

## 功能特性

- **内置打印机支持**: 支持POS内置热敏/标签打印机
- **蓝牙打印机支持**: 支持蓝牙连接外部打印机
- **跨平台调用**: Web页面、Android应用、React Native、Flutter等
- **标准打印框架**: 基于Android Print Framework

## 安装配置

### 1. 安装应用

```bash
adb install PrinterService_v1.0.A5_XXXXXX_release.apk
```

### 2. 启用打印服务

1. 安装APK后找到 **PrinterService** 并打开
2. 授予必要权限

### 3. 配置打印机

#### 内置打印机 (POS)

1. 打开 PrinterService 应用
2. 选择设备类型为 **POS (内置)**
3. 选择打印模式（标签/热敏）
4. 配置纸张参数并保存

#### 蓝牙打印机

1. 打开 PrinterService 应用
2. 选择设备类型为 **蓝牙**
3. 通过以下方式之一获取打印机 MAC 地址：
   - 点击 **扫码输入** 扫描打印机二维码
   - 点击 **搜索设备** 扫描并选择蓝牙设备
   - 手动输入 MAC 地址 (格式: `AA:BB:CC:DD:EE:FF`)
4. 配置纸张类型和指令类型
5. 点击 **连接蓝牙** 测试连接
6. 保存配置

## 使用场景

### 场景 1: 网页调用打印

网页可以通过标准的 Web Print API 调用打印服务。

#### JavaScript 示例

```javascript
// 方法 1: 使用 window.print()
function printPage() {
    window.print();
}

// 方法 2: 打印特定内容
function printContent(elementId) {
    const content = document.getElementById(elementId);
    const printWindow = window.open('', '', 'height=600,width=800');
    
    printWindow.document.write('<html><head><title>打印</title>');
    printWindow.document.write('</head><body>');
    printWindow.document.write(content.innerHTML);
    printWindow.document.write('</body></html>');
    
    printWindow.document.close();
    printWindow.print();
}

// 方法 3: 打印 PDF 文件
function printPDF(pdfUrl) {
    const iframe = document.createElement('iframe');
    iframe.style.display = 'none';
    iframe.src = pdfUrl;
    document.body.appendChild(iframe);
    
    iframe.onload = function() {
        iframe.contentWindow.print();
    };
}
```

#### HTML 示例

```html
<!DOCTYPE html>
<html>
<head>
    <title>打印测试</title>
    <style>
        @media print {
            /* 打印样式 */
            .no-print { display: none; }
            body { margin: 0; padding: 10px; }
        }
    </style>
</head>
<body>
    <div id="printArea">
        <h1>订单详情</h1>
        <p>订单号: 123456789</p>
        <p>金额: ¥100.00</p>
    </div>
    
    <button class="no-print" onclick="window.print()">打印</button>
</body>
</html>
```

### 场景 2: Android 应用调用打印

#### 方法 1: 使用 Android Print Framework

```java
import android.content.Context;
import android.print.PrintAttributes;
import android.print.PrintDocumentAdapter;
import android.print.PrintManager;
import android.webkit.WebView;
import android.webkit.WebViewClient;

public class PrintHelper {
    
    /**
     * 打印网页内容
     */
    public static void printWebPage(Context context, String htmlContent) {
        WebView webView = new WebView(context);
        webView.setWebViewClient(new WebViewClient() {
            @Override
            public void onPageFinished(WebView view, String url) {
                createPrintJob(context, view);
            }
        });
        
        webView.loadDataWithBaseURL(null, htmlContent, "text/html", "UTF-8", null);
    }
    
    /**
     * 创建打印任务
     */
    private static void createPrintJob(Context context, WebView webView) {
        PrintManager printManager = (PrintManager) context.getSystemService(Context.PRINT_SERVICE);
        
        String jobName = "Document_" + System.currentTimeMillis();
        PrintDocumentAdapter printAdapter = webView.createPrintDocumentAdapter(jobName);
        
        PrintAttributes attributes = new PrintAttributes.Builder()
                .setMediaSize(PrintAttributes.MediaSize.ISO_A4)
                .setResolution(new PrintAttributes.Resolution("id", "label", 203, 203))
                .setMinMargins(PrintAttributes.Margins.NO_MARGINS)
                .build();
        
        printManager.print(jobName, printAdapter, attributes);
    }
}
```

#### 方法 2: 打印自定义视图

```java
import android.graphics.Canvas;
import android.graphics.pdf.PdfDocument;
import android.print.PrintAttributes;
import android.print.pdf.PrintedPdfDocument;

public class CustomViewPrinter {
    
    /**
     * 打印自定义视图
     */
    public static void printView(Context context, View view) {
        PrintManager printManager = (PrintManager) context.getSystemService(Context.PRINT_SERVICE);
        
        String jobName = "CustomView_" + System.currentTimeMillis();
        
        printManager.print(jobName, new PrintDocumentAdapter() {
            @Override
            public void onLayout(PrintAttributes oldAttributes, 
                                PrintAttributes newAttributes,
                                CancellationSignal cancellationSignal,
                                LayoutResultCallback callback,
                                Bundle extras) {
                
                if (cancellationSignal.isCanceled()) {
                    callback.onLayoutCancelled();
                    return;
                }
                
                PrintDocumentInfo info = new PrintDocumentInfo.Builder(jobName)
                        .setContentType(PrintDocumentInfo.CONTENT_TYPE_DOCUMENT)
                        .setPageCount(1)
                        .build();
                
                callback.onLayoutFinished(info, true);
            }
            
            @Override
            public void onWrite(PageRange[] pages,
                               ParcelFileDescriptor destination,
                               CancellationSignal cancellationSignal,
                               WriteResultCallback callback) {
                
                PdfDocument pdfDocument = new PdfDocument();
                PdfDocument.PageInfo pageInfo = new PdfDocument.PageInfo.Builder(
                        view.getWidth(), view.getHeight(), 1).create();
                
                PdfDocument.Page page = pdfDocument.startPage(pageInfo);
                view.draw(page.getCanvas());
                pdfDocument.finishPage(page);
                
                try {
                    pdfDocument.writeTo(new FileOutputStream(destination.getFileDescriptor()));
                    callback.onWriteFinished(new PageRange[]{PageRange.ALL_PAGES});
                } catch (IOException e) {
                    callback.onWriteFailed(e.toString());
                } finally {
                    pdfDocument.close();
                }
            }
        }, null);
    }
}
```

#### 方法 3: 打印位图

```java
import android.graphics.Bitmap;
import android.graphics.Canvas;
import android.graphics.pdf.PdfDocument;

public class BitmapPrinter {
    
    /**
     * 打印位图
     */
    public static void printBitmap(Context context, Bitmap bitmap) {
        PrintManager printManager = (PrintManager) context.getSystemService(Context.PRINT_SERVICE);
        
        String jobName = "Bitmap_" + System.currentTimeMillis();
        
        printManager.print(jobName, new PrintDocumentAdapter() {
            @Override
            public void onLayout(PrintAttributes oldAttributes,
                                PrintAttributes newAttributes,
                                CancellationSignal cancellationSignal,
                                LayoutResultCallback callback,
                                Bundle extras) {
                
                PrintDocumentInfo info = new PrintDocumentInfo.Builder(jobName)
                        .setContentType(PrintDocumentInfo.CONTENT_TYPE_PHOTO)
                        .setPageCount(1)
                        .build();
                
                callback.onLayoutFinished(info, true);
            }
            
            @Override
            public void onWrite(PageRange[] pages,
                               ParcelFileDescriptor destination,
                               CancellationSignal cancellationSignal,
                               WriteResultCallback callback) {
                
                PdfDocument pdfDocument = new PdfDocument();
                PdfDocument.PageInfo pageInfo = new PdfDocument.PageInfo.Builder(
                        bitmap.getWidth(), bitmap.getHeight(), 1).create();
                
                PdfDocument.Page page = pdfDocument.startPage(pageInfo);
                Canvas canvas = page.getCanvas();
                canvas.drawBitmap(bitmap, 0, 0, null);
                pdfDocument.finishPage(page);
                
                try {
                    pdfDocument.writeTo(new FileOutputStream(destination.getFileDescriptor()));
                    callback.onWriteFinished(new PageRange[]{PageRange.ALL_PAGES});
                } catch (IOException e) {
                    callback.onWriteFailed(e.toString());
                } finally {
                    pdfDocument.close();
                }
            }
        }, null);
    }
}
```

### 场景 3: 跨应用调用

#### 通过 Intent 调用打印

```java
import android.content.Intent;
import android.net.Uri;

public class CrossAppPrinter {
    
    /**
     * 打印文件（PDF、图片等）
     */
    public static void printFile(Context context, Uri fileUri) {
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setDataAndType(fileUri, "application/pdf");
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        
        // 触发打印对话框
        if (intent.resolveActivity(context.getPackageManager()) != null) {
            context.startActivity(intent);
        } else {
            Toast.makeText(context, "未找到可用的打印服务", Toast.LENGTH_SHORT).show();
        }
    }
    
    /**
     * 分享到打印服务
     */
    public static void shareToPrint(Context context, String text) {
        Intent shareIntent = new Intent(Intent.ACTION_SEND);
        shareIntent.setType("text/plain");
        shareIntent.putExtra(Intent.EXTRA_TEXT, text);
        
        Intent chooser = Intent.createChooser(shareIntent, "选择打印方式");
        context.startActivity(chooser);
    }
}
```

### 场景 4: React Native / Flutter 集成

#### React Native 示例

```javascript
// 使用 react-native-print 库
import RNPrint from 'react-native-print';

async function printHTML() {
    const html = `
        <html>
            <body>
                <h1>订单详情</h1>
                <p>订单号: 123456789</p>
            </body>
        </html>
    `;
    
    await RNPrint.print({ html });
}

async function printPDF() {
    await RNPrint.print({
        filePath: '/path/to/document.pdf'
    });
}
```

#### Flutter 示例

```dart
// 使用 printing 包
import 'package:printing/printing.dart';
import 'package:pdf/pdf.dart';
import 'package:pdf/widgets.dart' as pw;

Future<void> printDocument() async {
  final pdf = pw.Document();
  
  pdf.addPage(
    pw.Page(
      build: (context) => pw.Center(
        child: pw.Text('Hello PrinterService!'),
      ),
    ),
  );
  
  await Printing.layoutPdf(
    onLayout: (format) => pdf.save(),
  );
}
```

## 配置参数说明

### POS打印模式

- **标签打印** (Label Mode): 适用于POS标签纸打印
- **热敏打印** (Thermal Mode): 适用于POS热敏纸打印

### 蓝牙打印机纸张类型

- **连续热敏纸** (Continuous Thermal Paper): 无间隙的连续纸张
- **间隙标签纸** (Label Paper): 带间隙的标签纸
- **黑标纸** (Black Mark Paper): 带黑色标记的纸张

### 蓝牙打印机指令类型

- **ESC 指令**: ESC/POS 标准指令集
- **CPCL 指令**: Common Printer Command Language

### 蓝牙打印机纸张尺寸

- **宽度** (Width): 纸张宽度（像素）
- **高度** (Height): 纸张高度（像素）

## 错误处理

### 常见错误码

| 错误码 | 说明 | 解决方案 |
|--------|------|----------|
| -96 | 蓝牙连接失败 | 检查蓝牙是否开启，MAC 地址是否正确 |
| -97 | MAC 地址无效 | 检查 MAC 地址格式 (AA:BB:CC:DD:EE:FF) |
| -98 | 不支持标签打印 | 切换到热敏打印模式 |
| -99 | 数据解析错误 | 检查打印内容格式 |
| PRNSTS_BUSY | 打印机忙碌 | 等待当前任务完成 |
| PRNSTS_OUT_OF_PAPER | 缺纸 | 添加打印纸 |
| PRNSTS_OVER_HEAT | 过热 | 等待打印机冷却 |
| PRNSTS_UNDER_VOLTAGE | 电压不足 | 检查电源连接 |

## 技术架构

```
┌─────────────────────────────────────┐
│      应用层 (Apps/WebView)          │
│   网页、Android App、跨平台应用    │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Android Print Framework           │
│   (PrintManager, PrintService API)  │
└──────────────┬──────────────────────┘
               │
               ▼
┌────────────────────────────────────┐
│      PrinterService                │
│  ┌─────────────────────────────┐   │
│  │  Printer (PrintService)     │   │
│  │  - onPrintJobQueued()       │   │
│  │  - PDF to Bitmap            │   │
│  └──────────┬──────────────────┘   │
│             │                      │
│             ▼                      │
│  ┌─────────────────────────────┐   │
│  │  PrinterFactory             │   │
│  └──────────┬──────────────────┘   │
│             │                      │
│      ┌──────┴──────┐               │
│      ▼             ▼               │
│  ┌────────┐   ┌─────────┐          │
│  │ Inner  │   │   BT    │          │
│  │Printer │   │ Printer │          │
│  └────┬───┘   └────┬────┘          │
└───────┼────────────┼───────────────┘
        │            │
        ▼            ▼
   ┌────────┐    ┌──────────┐
   │ 内置   │    │  蓝牙    │
   │打印机  │    │ 打印机   │
   └────────┘    └──────────┘
```

## 注意事项

1. **服务安装**: APK需预装或用户手动安装
2. **权限要求**: 蓝牙打印需要蓝牙权限
3. **标准框架**: 所有打印调用都通过Android标准Print Framework
4. **内容格式**: 服务接收的是PDF格式的打印文档
5. **配置优先**: 在PrinterService应用中配置好打印机后才能正常打印

## 技术支持

如需技术支持,请联系UROVO技术支持团队。

## License

Copyright © UROVO Technology Co., Ltd.
