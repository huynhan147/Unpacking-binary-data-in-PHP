# Unpacking binary data in PHP

Working with binary files in PHP is rarely a requirement. However when needed the PHP 'pack' and 'unpack' functions can help you tremendously. To set the stage we will start with a programming problem, this will keep the discussion anchored to a relevant context. The problem is this : We want to write a function that takes a image file as an argument and tells us whether the file is a GIF image; irrelevant with whatever the extension the file may have. We are not to use any GD library functions.  

#### A GIF file header

With the requirement that we are not allowed to use any graphics functions, to solve the problem we need to get the relevant data from the GIF file itself. Unlike a HTML or XML or other text format files, a GIF file and most other image formats are stored in a binary format. Most binary files carry a header at the top of the file which provides the meta information regarding the particular file. We can use this information to find out the type of the file and other things, such as height an width in case of a GIF file. A typical raw GIF header is shown below, using a hex editor such as [WinHex][1]. 

![](http://www.codediesel.com/wp-content/uploads/2010/09/winhex.gif)

The detailed description of the header is given below.


    
    
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



So to check if the image file is a valid GIF, we need to check the starting 3 bytes of the header, which have the 'GIF' marker, and the next 3 bytes, which give the version number; either '87a' or '89a'. It is for tasks such as these that the unpack() function is indispensable. Before we look at the solution, we will take a quick look at the unpack() function itself.

#### Using the unpack() function

[unpack()][3] is the complement of [pack()][4] – it transforms binary data into an associative array based on the format specified. It is somewhat along the lines of _sprintf_, transforming string data according to some given format. These two functions allow us to read and write buffers of binary data according to a specified format string. This easily enables a programmer to exchange data with programs written in other languages or other formats. Take a small example.


    
    
    $data = unpack('C*', 'codediesel');
    var_dump($data);


This will print the following, decimal codes for 'codediesel' :

    
    
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


In the above example the first argument is the format string and the second the actual data. The format string specifies how the data argument should be parsed. In this example the first part of the format 'C', specifies that we should treat the first character of the data as a unsigned byte. The next part '*', tells the function to apply the previously specified format code to all the remaining characters.

Although this may look confusing, the next section provides a concrete example.

#### Grabbing the header data

Below is the solution to our GIF problem using the unpack() function. The _is_gif()_ function will return true if the given file is in a GIF format.

    
    
    function is_gif($image_file)
    {
     
        /* Open the image file in binary mode */
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



The important line to note is the format specifier. The 'A6' characters specifies that the unpack() function is to get the the first 6 bytes of the data and interpret it as a string. The retrieved data is then stored in an associate array with the key named 'version'.

Another example is given below. This returns some additional header data of the GIF file, including the image width and height.

    
    
    function get_gif_header($image_file)
    {
     
        /* Open the image file in binary mode */
        if(!$fp = fopen ($image_file, 'rb')) return 0;
     
        /* Read 20 bytes from the top of the file */
        if(!$data = fread ($fp, 20)) return 0;
     
        /* Create a format specifier */
        $header_format = 
                'A6Version/' . # Get the first 6 bytes
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

 | 

The above example will print the following when run.

    
    
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

 

Below we will go into the details of how the format specifier works. I'll split the format, giving the details for each character.


    
    
    $header_format = 'A6Version/C2Width/C2Height/C1Flag/@11/C1Aspect';
  
    
    A - Read a byte and interpret it as a string. 
        Number of bytes to read is given next
    6 - Read a total of 6 bytes, starting from position 0
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

 

More format options can be found [here][4]. Although I've only shown a small example, the pack/unpack is capable of much complex work than presented here.

Note: Since PHP version 7.2.0 float and double types supports both Big Endian and Little Endian.
