# [Link ImageChecker](https://packagist.org/packages/huyphi/image_checker)  
# Giải nén dữ liệu nhị phân trong PHP

Rất ít khi chúng ta phải làm việc với các file nhị nhân trong PHP. Tuy nhiên khi cần thì hàm 'pack' và 'unpack' trong PHP có thể giúp bạn rất nhiều. Để bắt đầu, chúng ta sẽ bắt đầu với một vấn đề lập trình, điều này sẽ giúp cho cuộc thảo luận được gắn với hoàn cảnh liên quan. Vấn đề ở đây là : Chúng ta muốn viết một hàm có đối số là một file ảnh và cho biết có phải là ảnh GIF hay không,bất kể file có đuôi như thế nào. Chúng ta không sử dụng bất kỳ hàm thư viện GD nào.

#### A GIF file header

Với yêu cầu là không sử dụng bất kỳ hàm đồ họa nào, để giải quyết vấn đề này chúng ta cần lấy dữ liệu liên quan từ chính file GIF. Không giống như file HTML hoặc XML hoặc các file định dạng văn bản khác, một file GIF và hầu hết các định dạng hình ảnh khác được lưu trữ ở định dạng nhị phân. Hầu hết các file nhị phân đều có header ở đầu file cung cấp thông tin meta về file cụ thể. Chúng ta có thể sử dụng thông tin này để tìm ra loại file và những thứ khác, chẳng hạn như chiều cao chiều rộng trong trường hợp là một file GIF. Một header GIF thông thường được hiển thị như bên dưới, sử dụng trình soạn thảo hex như [WinHex](1). 

![](http://www.codediesel.com/wp-content/uploads/2010/09/winhex.gif)

Mô tả chi tiết header ở bên dưới


    
    
    Offset   Length   Contents
      0      3 bytes  "GIF"
      3      3 bytes  "87a" or "89a"
      6      2 bytes  
      8      2 bytes  
     10      1 byte   bit 0:    Global Color Table Flag (GCTF)
                      bit 1..3: Color Resolution
                      bit 4:    Sort Flag to Global Color Table
                      bit 5..7: Size of Global Color Table: 2^(1+n)
     11      1 byte   
     12      1 byte   
     13      ? bytes  
             ? bytes  
             1 bytes   (0x3b)



Vì vậy để kiểm tra một file ảnh có đúng là một file GIF không, Chúng ta cần phải kiểm tra 3 byte đầu của phần header, có 'GIF', và 3 byte tiếp theo, là số phiên bản '87a' hoặc '89a'. Nó là những thứ mà hàm unpack() cần phải thực hiện. Trước khi chúng ta tìm giải pháp, nhìn qua xem hàm unpack() hoạt động thế nào.

#### Sử dụng hàm unpack()

[unpack()](3) là sự bổ sung của [pack()](4) – nó chuyển đổi dữ liệu nhị phân thành một mảng liên kết dựa trên định dạng được chỉ định. Nó có phần giống với _sprintf_, Chuyển đổi chuôĩ dữ liệu theo một số định dạng nhất định. Hai hàm này cho phép chúng ta đọc và ghi các bộ đệm của dữ liệu nhị phân theo một chuỗi định dạng được chỉ định. Điều này cho phép một lập trình viên dễ dàng chuyển đổi dữ liệu với các chương trình được viết bằng các ngôn ngữ khác hoặc các định dạng khác. Lấy một ví dụ nhỏ.



    
    
    $data = unpack('C*', 'codediesel');
    var_dump($data);


Thao tác này sẽ in các mã thập phân của 'codediesel' :

    
    
    array
      1 => int 99
      2 => int 111
      3 => int 100
      4 => int 101
      5 => int 100
      6 => int 105
      7 => int 101
      8 => int 115
      9 => int 101
      10 => int 108


Trong ví dụ trên, đối số đầu tiên là chuỗi định dạng và thứ hai là dữ liệu thực tế.Chuỗi định dạng xác định cách parse đối số dữ liệu. Trong ví dụ này, phần đầu tiên của định dạng 'C', cho biết chúng ta nên xử lý ký tự đầu tiên của dữ liệu dưới dạng một unsigned byte. Phần tiếp theo ' * ', yêu cầu hàm áp dụng code định dạng được chỉ định trước đó cho tất cả các ký tự còn lại.

Mặc dù điều này có vẻ khó hiểu, phần tiếp theo cung cấp một ví dụ cụ thể.
#### Lấy dữ liệu header

Dưới đây là giải pháp cho vấn đề GIF của chúng ta bằng cách sử dụng hàm unpack (). Hàm _is_gif()_ sẽ trả về true nếu file đã cho ở định dạng GIF.

    
    
    function is_gif($image_file)
    {
     
        /* Mở file hình ảnh ở chế độ nhị phân */
        if(!$fp = fopen ($image_file, 'rb')) return 0;
     
        /* Read 20 bytes from the top of the file */
        if(!$data = fread ($fp, 20)) return 0;
     
        /* Create a format specifier */
        $header_format = 'A6version';  # Get the first 6 bytes
    
        /* Unpack the header data */
        $header = unpack ($header_format, $data);
     
        $ver = $header['version'];
     
        return ($ver == 'GIF87a' || $ver == 'GIF89a')? true : false;
     
    }
     
    /* Run our example */
    echo is_gif("aboutus.gif");



Dòng quan trọng cần lưu ý là dòng format specifier. Ký tự 'A6' cho biết rằng hàm unpack() sẽ lấy 6 byte đầu tiên của dự liệu và xuất nó ra dưới dạng chuỗi. Dữ liệu đã truy xuất sau đó được lưu trữ trong một mảng liên kết với khóa có tên là 'version'.

Một ví dụ khác được đưa ra dưới đây. Nó trả về một số dữ liệu bổ sung trong header của file GIF bao gồm chiều rộng và chiều cao của ảnh.

    
    
    function get_gif_header($image_file)
    {
     
        /* Open the image file in binary mode */
        if(!$fp = fopen ($image_file, 'rb')) return 0;
     
        /* Read 20 bytes from the top of the file */
        if(!$data = fread ($fp, 20)) return 0;
     
        /* Create a format specifier */
        $header_format = 
                'A6Version/' . # Lấy 6 bytes đầu tiên
                'C2Width/' .   # Get the next 2 bytes
                'C2Height/' .  # Get the next 2 bytes
                'C1Flag/' .    # Get the next 1 byte
                '@11/' .       # Jump to the 12th byte
                'C1Aspect';    # Get the next 1 byte
    
        /* Unpack the header data */
        $header = unpack ($header_format, $data);
     
        $ver = $header['Version'];
     
        if($ver == 'GIF87a' || $ver == 'GIF89a') {
            return $header;
        } else {
            return 0;
        }
    }
     
    /* Run our example */
    print_r(get_gif_header("aboutus.gif"));
 

Ví dụ trên khi chạy sẽ in ra như sau

    
    
    Array
    (
        [Version] => GIF89a
        [Width1] => 97
        [Width2] => 0
        [Height1] => 33
        [Height2] => 0
        [Flag] => 247
        [Aspect] => 0
    )

 

Dưới đây chúng ta sẽ đi vào chi tiết cách mà format specifier hoạt động. Tôi sẽ chi nhỏ các định dạng, đưa ra các chi tiết cho mỗi định dạng.


    
    
    $header_format = 'A6Version/C2Width/C2Height/C1Flag/@11/C1Aspect';
  
    
    A - Read a byte and interpret it as a string. 
        Number of bytes to read is given next
    6 - Đọc tổng cộng 6 bytes, bắt đầu từ vị trí 0
    Version - Name of key in the associative array where data 
        retrieved by 'A6' is stored
     
    / - Start a new code format
    C - Interpret the next data as an unsigned byte
    2 - Read a total of 2 bytes
    Width - Key in the associative array
     
    / - Start a new code format
    C - Interpret the data as an unsigned byte
    2 - Read a total of 2 bytes
    Height- Key in the associative array
     
    / - Start a new code format
    C - Interpret the data as an unsigned byte
    1 - Read a total of 2 bytes
    Flag - Key in the associative array
     
    / - Start a new code format
    @ - Move to the byte offset specified by the following number.
          Remember that the first position in the binary string is 0. 
    11 - Move to position 11
     
    / - Start a new code format
    C - Interpret the data as an unsigned byte
    1 - Read a total of 1 bytes
    Aspect - Key in the associative array

 

Bạn có thể tìm thấy các tùy chọn định dạng khác [tại đây](4). Mặc dù tôi chỉ trình bày một ví dụ nhỏ, pack/unpacka có thể làm được nhiều thứ hơn so với những cái tôi trình bày ở đây.

Note: Từ phiên bản PHP 7.2.0 kiểu float và double được hỗ trợ bởi cả Big Endian và Little Endian.
