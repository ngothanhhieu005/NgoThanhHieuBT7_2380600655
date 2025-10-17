using BUSS;
using DAL.Entities;
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.IO;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace NgoThanhHieuBTT7
{

    public partial class Form1 : Form
    {

        private readonly StudentService studentService = new StudentService();
        private readonly FacultyService facultyService = new FacultyService();
        private string avatarFilePath = string.Empty; // Khai báo biến toàn cục
        public Form1()
        {
            InitializeComponent();
            // Gán sự kiện CellClick cho DataGridView
            this.dgvStudent.CellClick += new DataGridViewCellEventHandler(this.dgvStudent_CellClick);
        }

        private void Form1_Load(object sender, EventArgs e)
        {
            try
            {
                setGridViewStyle(dgvStudent);
                var listFacultys = facultyService.GetAll();
                var listStudents = studentService.GetAll();
                FillFalcultyCombobox(listFacultys);
                BindGrid(listStudents);
                ClearData(); // Đảm bảo form sạch sẽ khi khởi động
            }
            catch (Exception ex)
            {
                MessageBox.Show(ex.Message);
            }
        }

        private void FillFalcultyCombobox(List<Faculty> listFacultys)
        {
            // Thêm mục "Chọn khoa" hoặc mục rỗng
            listFacultys.Insert(0, new Faculty() { FacultyID = 0, FacultyName = "" });
            this.cmbFaculty.DataSource = listFacultys;
            this.cmbFaculty.DisplayMember = "FacultyName";
            this.cmbFaculty.ValueMember = "FacultyID";
        }

        private void BindGrid(List<Student> listStudent)
        {
            dgvStudent.Rows.Clear();
            foreach (var item in listStudent)
            {
                int index = dgvStudent.Rows.Add();
                dgvStudent.Rows[index].Cells[0].Value = item.StudentID;
                dgvStudent.Rows[index].Cells[1].Value = item.FullName;
                // Hiển thị Tên Khoa
                if (item.Faculty != null)
                    dgvStudent.Rows[index].Cells[2].Value = item.Faculty.FacultyName;
                // Hiển thị Điểm
                dgvStudent.Rows[index].Cells[3].Value = item.AverageScore.ToString();
                // Hiển thị Chuyên Ngành (Major)
                if (item.MajorID != null && item.Major != null)
                    dgvStudent.Rows[index].Cells[4].Value = item.Major.Name;

                // Không gọi ShowAvatar ở đây vì nó sẽ chỉ hiển thị avatar của sinh viên cuối cùng
                // Thay vào đó, nó sẽ được gọi khi người dùng click vào DataGridView
            }
        }

        // Phương thức này hiện tại không được sử dụng hiệu quả, nên thay thế bằng LoadAvatar
        private void ShowAvatar(string avatar)
        {
            // Bỏ trống hoặc xóa phương thức này
        }

        public void setGridViewStyle(DataGridView dgview)
        {
            dgview.BorderStyle = BorderStyle.None;
            dgview.DefaultCellStyle.SelectionBackColor = Color.DarkTurquoise;
            dgview.CellBorderStyle = DataGridViewCellBorderStyle.SingleHorizontal;
            dgview.BackgroundColor = Color.White;
            dgview.SelectionMode = DataGridViewSelectionMode.FullRowSelect;
        }

        private void pictureBox1_Click(object sender, EventArgs e)
        {
            // Giữ nguyên
        }

        // Đã hợp nhất với mã cũ của bạn
        private void btnUpLoad_Click(object sender, EventArgs e)
        {
            using (OpenFileDialog openFileDialog = new OpenFileDialog())
            {
                openFileDialog.Filter = "Image Files (*.jpg; *.jpeg; *.png)|*.jpg; *.jpeg; *.png";
                if (openFileDialog.ShowDialog() == DialogResult.OK)
                {
                    avatarFilePath = openFileDialog.FileName;
                    picAvatar.Image = Image.FromFile(avatarFilePath);
                }
            }
        }

        private void LoadAvatar(string studentID)
        {
            string folderPath = Path.Combine(Application.StartupPath, "Images");
            // Cast the result to Student, since FindStudentByID returns object
            var studentObj = studentService.FindStudentByID(studentID);
            var student = studentObj as Student;

            if (student != null && !string.IsNullOrEmpty(student.Avatar))
            {
                string avatarFilePath = Path.Combine(folderPath, student.Avatar);
                if (File.Exists(avatarFilePath))
                {
                    if (picAvatar.Image != null) picAvatar.Image.Dispose();
                    using (var stream = new FileStream(avatarFilePath, FileMode.Open, FileAccess.Read))
                    {
                        picAvatar.Image = Image.FromStream(stream);
                    }
                }
                else
                {
                    picAvatar.Image = null;
                }
            }
            else
            {
                picAvatar.Image = null;
            }
        }

        private string SaveAvatar(string sourceFilePath, string studentID)
        {
            try
            {
                string folderPath = Path.Combine(Application.StartupPath, "Images");
                if (!Directory.Exists(folderPath))
                {
                    Directory.CreateDirectory(folderPath);
                }

                string fileExtension = Path.GetExtension(sourceFilePath);
                string targetFilePath = Path.Combine(folderPath, $"{studentID}{fileExtension}");

                if (!File.Exists(sourceFilePath))
                {
                    throw new FileNotFoundException($"Không tìm thấy file: {sourceFilePath}");
                }

                // File.Copy(sourceFilePath, targetFilePath, true);
                // Để tránh lỗi "The process cannot access the file..." nếu file đang được picAvatar sử dụng,
                // ta nên sử dụng File.Delete() trước hoặc dùng FileStream
                if (File.Exists(targetFilePath))
                {
                    File.Delete(targetFilePath);
                }

                File.Copy(sourceFilePath, targetFilePath, true);

                return $"{studentID}{fileExtension}";
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error saving avatar: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return null;
            }
        }

        private void btnAdd_Click(object sender, EventArgs e)
        {
            try
            {
                // **Kiểm tra và báo lỗi đầu vào**
                if (!ValidateInput()) return;

                // **Tìm sinh viên: Dùng ID.Text (Tên điều khiển mới của bạn)**
                var studentObj = studentService.FindStudentByID(ID.Text);
                var student = studentObj as Student ?? new Student();

                // Update student details
                student.StudentID = ID.Text;
                student.FullName = FullName.Text;
                student.AverageScore = double.Parse(DTB.Text);

                // Kiểm tra SelectedValue có null không trước khi Parse
                if (cmbFaculty.SelectedValue != null)
                {
                    student.FacultyID = int.Parse(cmbFaculty.SelectedValue.ToString());
                }

                // Check if an avatar file has been selected (Kiểm tra biến toàn cục)
                if (!string.IsNullOrEmpty(avatarFilePath))
                {
                    string avatarFileName = SaveAvatar(avatarFilePath, ID.Text);
                    if (!string.IsNullOrEmpty(avatarFileName))
                    {
                        student.Avatar = avatarFileName;
                    }
                }

                // Nếu sinh viên đã có Avatar cũ và người dùng không upload cái mới, thì giữ nguyên.
                // Nếu người dùng upload cái mới, nó đã được cập nhật ở khối IF trên.

                // **Thực hiện Thêm/Sửa**
                studentService.InsertUpdate(student); // Dùng InsertUpdate theo StudentService cũ của bạn

                // **Làm mới dữ liệu**
                BindGrid(studentService.GetAll());

                // **Xóa dữ liệu trên Form và reset đường dẫn**
                ClearData();
                MessageBox.Show("Cập nhật dữ liệu sinh viên thành công!");
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Lỗi khi thêm/sửa dữ liệu: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        // --- BỔ SUNG CÁC PHƯƠNG THỨC HỖ TRỢ ---

        // Phương thức tải dữ liệu sinh viên lên các điều khiển khi click vào DataGridView
        private void dgvStudent_CellClick(object sender, DataGridViewCellEventArgs e)
        {
            if (e.RowIndex >= 0 && e.RowIndex < dgvStudent.Rows.Count - 1) // Tránh hàng header và hàng trống cuối cùng
            {
                // Lấy StudentID từ cột đầu tiên (Cells[0])
                string studentID = dgvStudent.Rows[e.RowIndex].Cells[0].Value.ToString();
                LoadDataForEdit(studentID);
            }
        }

        private void LoadDataForEdit(string studentID)
        {
            var studentObj = studentService.FindStudentByID(studentID);
            var student = studentObj as Student;
            if (student != null)
            {
                // **Sử dụng các tên điều khiển mới của bạn**
                ID.Text = student.StudentID;
                FullName.Text = student.FullName;
                DTB.Text = student.AverageScore.ToString();

                // Chọn đúng khoa trong Combobox
                if (student.FacultyID != 0)
                {
                    cmbFaculty.SelectedValue = student.FacultyID;
                }

                // Tải ảnh đại diện
                LoadAvatar(student.StudentID);

                // Reset avatarFilePath để chỉ khi có upload mới thì mới lưu
                avatarFilePath = string.Empty;
            }
        }


        // Phương thức kiểm tra đầu vào (Đã điều chỉnh tên điều khiển)
        private bool ValidateInput()
        {
            if (string.IsNullOrWhiteSpace(ID.Text))
            {
                MessageBox.Show("Mã Sinh Viên không được để trống.", "Lỗi", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return false;
            }
            if (string.IsNullOrWhiteSpace(FullName.Text))
            {
                MessageBox.Show("Họ Tên không được để trống.", "Lỗi", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return false;
            }
            if (!double.TryParse(DTB.Text, out double score) || score < 0 || score > 10)
            {
                MessageBox.Show("Điểm Trung Bình phải là số hợp lệ từ 0 đến 10.", "Lỗi", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return false;
            }
            if (cmbFaculty.SelectedValue == null || (int)cmbFaculty.SelectedValue == 0)
            {
                MessageBox.Show("Vui lòng chọn Khoa.", "Lỗi", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return false;
            }
            return true;
        }

        // Phương thức xóa dữ liệu trên Form (Đã điều chỉnh tên điều khiển)
        private void ClearData()
        {
            ID.Text = string.Empty;
            FullName.Text = string.Empty;
            DTB.Text = string.Empty;
            cmbFaculty.SelectedIndex = 0;
            picAvatar.Image = null;
            avatarFilePath = string.Empty; // Reset đường dẫn file tạm
        }
    }
}
