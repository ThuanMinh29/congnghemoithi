const Product = require('../models/product');
const AWS = require('aws-sdk');
const multer = require('multer');
const { v4: uuidv4 } = require('uuid');
const dotenv = require('dotenv');

// Đọc các thông tin từ file .env
dotenv.config();

// Cấu hình AWS SDK với thông tin từ .env
AWS.config.update({
  accessKeyId: process.env.ACCESS_KEY,        // Lấy từ .env
  secretAccessKey: process.env.SECRET_KEY,  // Lấy từ .env
  region: process.env.REGION,                     // Lấy từ .env
});
const CLOUD_FRONT_URL = 'https://do9rg35qf6giw.cloudfront.net'; // Lấy ở Detail của CloudFront
// Cấu hình multer để upload file
const storage = multer.memoryStorage();
const upload = multer({
  storage,
  limits: { fileSize: 2 * 1024 * 1024 },  // Giới hạn dung lượng file là 2MB
  fileFilter(req, file, cb) {
    const fileTypes = /jpeg|jpg|png|gif/;
    const extname = fileTypes.test(file.originalname.split('.').pop());
    const mimetype = fileTypes.test(file.mimetype);
    if (extname && mimetype) {
      return cb(null, true);
    }
    return cb("Error: Image Only");
  },
});

// Hiển thị danh sách sản phẩm
const showProducts = async (req, res) => {
    try {
      const sanPhams = await Product.getAllProducts();
      res.render('index', { sanPhams });
    } catch (err) {
      console.error('Error fetching products:', err); // Log lỗi khi lấy sản phẩm
      res.status(500).send('Internal Server Error');
      
    }
  };
  // Thêm sản phẩm
const addProduct = async (req, res) => {

    const { ma_sp, ten_sp, so_luong } = req.body;
    let image_url = '';
  
    if (req.file) {
      const image = req.file.originalname.split(".");
      const fileType = image[image.length - 1];
      const filePath = `${uuidv4()}-${Date.now().toString()}.${fileType}`;
  
      const s3 = new AWS.S3();
      const params = {
        Bucket: process.env.BUCKET_NAME,
        Key: filePath,
        Body: req.file.buffer,
        ContentType: req.file.mimetype,
      };
  
      s3.upload(params, async (error, data) => {
        if (error) {
          console.log('S3 upload error:', error);  // Log lỗi của S3
          return res.send('Internal Server Error'); //log

        } else {
          image_url = `${CLOUD_FRONT_URL}${filePath}`;
          try {
            await Product.addProduct(ma_sp, ten_sp, so_luong, image_url);
            res.redirect('/');
          } catch (err) {
            res.send('Internal Server Error');
          }
        }
      });
    } else {
      try {
        await Product.addProduct(ma_sp, ten_sp, so_luong);
        res.redirect('/');
      } catch (err) {
        res.send('Internal Server Error');
      }
    }
  };
  // Xoá sản phẩm
const deleteProduct = async (req, res) => {
    let { ma_sp } = req.body;
  
    if (!ma_sp) return res.redirect('/');
    if (!Array.isArray(ma_sp)) ma_sp = [ma_sp];
  
    for (let id of ma_sp) {
      try {
        await Product.deleteProduct(id);
      } catch (err) {
        return res.send('Xoá thất bại');
      }
    }
    res.redirect('/');
  };
  module.exports = { showProducts, addProduct, deleteProduct, upload };





  <!DOCTYPE html>
<html lang="en">
  <head>
    <title>Title</title>
    <!-- Required meta tags -->
    <meta charset="utf-8" />
    <meta
      name="viewport"
      content="width=device-width, initial-scale=1, shrink-to-fit=no"
    />

    <!-- Bootstrap CSS v5.2.1 -->
    <link
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css"
      rel="stylesheet"
      integrity="sha384-T3c6CoIi6uLrA9TneNEoa7RxnatzjcDSCmG1MXxSR1GAsXEV/Dwwykc2MPK8M2HN"
      crossorigin="anonymous"
    />

    <script>
      function validateForm() {
        const ten_sp = document.forms["addProduct"]["ten_sp"].value;
  
        // Kiểm tra tên sản phẩm không để trống và không chứa ký tự đặc biệt
        const namePattern = /^[a-zA-Z0-9\s]+$/; // Chỉ cho phép chữ cái, số và dấu cách
        if (!ten_sp || !namePattern.test(ten_sp)) {
          alert("Tên sản phẩm không hợp lệ. Không có ký tự đặc biệt và không được để trống.");
          return false; // Ngừng gửi form
        }
  
        return true; // Nếu tất cả hợp lệ, form sẽ được gửi
      }
    </script>

  </head>

  <body>
    <div>
      <div>
        <div>
          <div class="mb-3">
            <form name="addProduct" action="/add" method="POST" enctype="multipart/form-data" class="form-input" onsubmit="return validateForm()">
              <input
                type="number"
                name="ma_sp"
                class="form-control"
                placeholder="Ma SP"
                required
              />
              <input
                type="text"
                name="ten_sp"
                class="form-control"
                placeholder="Ten SP"
              />
              <input
                type="number"
                name="so_luong"
                class="form-control"
                placeholder="So luong"
                min="0"
              />
              <input
                type="file"
                name="image"
                class="form-control"
                placeholder="Image"
              />
              <button type="submit">Submit</button>
            </form>
            
          </div>
        </div>
        <div class="table-responsive">
       <form  action="/delete" method="POST">
        <button type="submit">Xoa</button>
        <table class="table table-primary">
          <thead>
            <tr>
              <th scope="col">Ma SP</th>
              <th scope="col">Ten SP</th>
              <th scope="col">So luong</th>
              <th scope="col">Image</th>
              <th scope="col">Action</th>
            </tr>
          </thead>
          <tbody>
            <% sanPhams.forEach(item => { %>
              <tr>
                <td><%= item.ma_sp %></td>
                <td><%= item.ten_sp %></td>
                <td><%= item.so_luong %></td>
                <td><img width="50px" src="<%= item.image_url %>" /></td> <!-- Hiển thị ảnh -->
                <td><input type="checkbox" name="ma_sp" value="<%= item.ma_sp %>"></td>
              </tr>
            <% }) %>
          </tbody>
        </table>
       </form>
        </div>
      </div>
    </div>
    <script
      src="https://cdn.jsdelivr.net/npm/@popperjs/core@2.11.8/dist/umd/popper.min.js"
      integrity="sha384-I7E8VVD/ismYTF4hNIPjVp/Zjvgyol6VFvRkX/vR+Vc4jQkC+hVqc2pM8ODewa9r"
      crossorigin="anonymous"
    ></script>

    <script
      src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.min.js"
      integrity="sha384-BBtl+eGJRgqQAUMxJ7pMwbEyER4l1g+O15P+16Ep7Q9Q+zqX6gSbd85u4mG4QzX+"
      crossorigin="anonymous"
    ></script>
  </body>
</html>
