## 先展示一下效果及最终的代码

![image.png](http://140.143.145.178:8090/upload/2020/12/image-0f00a8bef88c4f46a8b6fc51e6df2dcc.png)
```
import cn.hutool.core.io.IoUtil;
import cn.hutool.poi.excel.ExcelUtil;
import cn.hutool.poi.excel.ExcelWriter;
import com.hnk.activity.constants.CheckStatusEnum;
import com.hnk.activity.excel.CheckDown;
import com.hnk.activity.input.DownSignUpInput;
import com.hnk.activity.repository.model.ActivityMember;
import com.hnk.activity.service.MapperService;
import io.swagger.annotations.ApiOperation;
import lombok.extern.slf4j.Slf4j;
import org.apache.poi.hssf.usermodel.HSSFClientAnchor;
import org.apache.poi.hssf.usermodel.HSSFPatriarch;
import org.apache.poi.hssf.usermodel.HSSFSheet;
import org.apache.poi.ss.usermodel.ClientAnchor;
import org.apache.poi.ss.usermodel.Workbook;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;

import javax.imageio.ImageIO;
import javax.servlet.ServletOutputStream;
import javax.servlet.http.HttpServletResponse;
import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.net.URL;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Controller
@RequestMapping("/file")
public class AttectmentControllerTemp {

    @Autowired
    private MapperService mapperService;

    @ApiOperation(value = "下载报名列表")
    @RequestMapping("/down/sign_up_file")
    public void getTransactionList(HttpServletResponse response, @RequestBody DownSignUpInput input) throws Exception {
     
        List<ActivityMember> members = mapperService.findActivityMemberByActivityMemberMidIn(input.getActivityMemberMids());
        int[] oroder = {1};

        List<CheckDown> rows = members.stream().map(r -> {
            CheckDown down = new CheckDown();
            down.setOrder(String.valueOf(oroder[0]++));
            down.setCreateAt(r.getCreateAt());
            down.setNickName(r.getNickName());
            down.setStatusString(CheckStatusEnum.getByCode(r.getStatus()).getLabel());
            return down;
        }).collect(Collectors.toList());

        ExcelWriter writer = ExcelUtil.getWriter();

        writer.addHeaderAlias("order", "序号");
        writer.addHeaderAlias("photo", "头像");
        writer.addHeaderAlias("nickName", "昵称");
        writer.addHeaderAlias("createAt", "报名时间");
        writer.addHeaderAlias("statusString", "审核状态");

        HSSFSheet sheet = (HSSFSheet) writer.getSheet();
        Workbook workbook = writer.getWorkbook();

        writer.write(rows, true);
        int row = 1;
        while (row <= oroder[0]) {
            writer.setRowHeight(row++, 35);
        }
        writer.setColumnWidth(1, 10);
        writer.setColumnWidth(2, 30);
        writer.setColumnWidth(3, 20);
        writer.merge(0, 0, 1, 2, "用户信息(微信头像和昵称)", true);

        int y = 0;
        for (ActivityMember r : members) {
            try {
                y++;
                URL url = new URL(r.getPhotoUrl());

                BufferedImage bufferedImage = ImageIO.read(url);
                ByteArrayOutputStream byteArrayOut = new ByteArrayOutputStream();
                ImageIO.write(bufferedImage, "jpg", byteArrayOut);
                byte[] data = byteArrayOut.toByteArray();
                HSSFPatriarch drawingPatriarch = sheet.createDrawingPatriarch();
                int x = 1;
                int width = 1;
                int height = 1;
                // anchor主要用于设置图片的属性
                HSSFClientAnchor anchor = new HSSFClientAnchor(0, 0, 0, 0, (short) x, y, (short) (x + width), (y + height));

                anchor.setAnchorType(ClientAnchor.AnchorType.byId(0));
                drawingPatriarch.createPicture(anchor, workbook.addPicture(data, XSSFWorkbook.PICTURE_TYPE_JPEG));

            } catch (Exception e) {

            }
        }

        String filename = new String(("报名信息-" + LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd")) + ".xls").getBytes("gb2312"), "ISO8859-1");

        response.setContentType("application/vnd.ms-excel;charset=utf-8");
        response.setHeader("Content-Disposition", "attachment;filename=" + filename);
        ServletOutputStream out = response.getOutputStream();

        writer.flush(out, true);
        writer.close();
        IoUtil.close(out);
    }

}
```


产品提了一个新需求，导出用户小程序端报名数据到excel。想想这还不简单，五分钟之内就可以解决上线。直到拿到原型，我才知道，大意了，竟然还要用户头像，导出图片这个之前没有搞过。

## 尝试开始
因为头像信息是微信返回的一个URL，第一个想法就是直接把URL放到excel中，看看能否显示。尝试了一下，直接放连接，加上src标签等等方式都没有成功。我看还有写vb的，那玩意不懂，而且excel要从服务器下载，这样整也不合适。

放弃了第一个想法后，没有思路，开始浏览网上各种博客，真的不得不说，csdn已经被做烂了，等我有空就把csdn给注销掉。终于在乱花从中找到了一点有用的东西，但是出处没有保存，抱歉
```
          Workbook wb = new HSSFWorkbook();
          Sheet sheet = wb.createSheet("图片");
          Drawing patriarch = sheet.createDrawingPatriarch();
          if (pictureUrl != null) {
                //写入流
                ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
                FileInputStream fileInputStream = new FileInputStream(new File(pictureUrl));
                byte[] data = new byte[1024];
                int len = 0;
                while ((len = fileInputStream.read(data)) != -1) {
                    outputStream.write(data, 0, len);
                }
                int x = 1;
                int y = 1;
                int width = 6;
                int height = 10;
                // anchor主要用于设置图片的属性
                HSSFClientAnchor anchor = new HSSFClientAnchor(0, 0, 255, 0, (short) x, y, (short) (x + width), (y + height));
                patriarch.createPicture(anchor, wb.addPicture(outputStream.toByteArray(), HSSFWorkbook.PICTURE_TYPE_JPEG));
            }
            OutputStream os = new FileOutputStream(new File("E:\\test.xls"));
            wb.write(os);
```
上面这一块呢就是导出图片到excel，图片定位在HSSFClientAnchor中设置。现在一张图片是导出了，但如何把图片加入到一条条数据中呢？
如果让我直接调poi去操作每个单元格，说实话从内心我是抗拒的，两三年前写那玩意，看着都眼花。这时候我想起了，平时很多工具类我都是用的[Hutool](Hutool)中的实现，安利一下这个github已经16.9K星的项目，的确很好。在Hutool中，果然找到了操作Excel的工具。看着Api搞起，Hutool支持List，Map和Bean等多种实现的导出。就我这个项目而言，果断选择对象导出。可以参考Hutool的API，写的很详细这里就跳过了。
![image.png](http://140.143.145.178:8090/upload/2020/12/image-cfe4fce659034b1fa8f77e60656ce3ab.png)
## 实现
既然能导出图片，能写出数据，现在就差把图片融合到数据里面了，而且要定位到图片该在的位置。
从导出图片的代码我们可以看到，HSSFPatriarch的createPicture方法中需要Workbook的对象，而HSSFPatriarch是由HSSFSheet生成的
```
HSSFPatriarch drawingPatriarch = sheet.createDrawingPatriarch();
drawingPatriarch.createPicture(anchor, workbook.addPicture(data, XSSFWorkbook.PICTURE_TYPE_JPEG));
```
Hutool已经对这些对象进行了封装，但是考虑到一个优秀的代码，应该会把封装的对象暴露出来，以供调用者可以自定义一些操作。果然在Hutool封装对象ExcelWriter中找到了getWorkbook和getSheet方法。然后把输出图片的代码用一个循环来导出每条数据的图片，调试一下行高列宽，执行。第一次导出数据，除了不太美观，基本功能已经实现，根据需求再调整一下样式，齐活。


最终的代码直接拷贝，替换调导出的数据就可以执行，对Hutool的引用可以参考官方文档。
