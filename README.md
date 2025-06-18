import zipfile
import os

# Tạo thư mục chứa mã nguồn
project_dir = "/mnt/data/AIVT_App_V9"
os.makedirs(project_dir, exist_ok=True)

# Mã nguồn chính để chạy Blockchain AIVT dưới dạng ứng dụng console
main_script = """
import hashlib, time, json

class Block:
    def __init__(self, index, timestamp, data, previous_hash):
        self.index = index
        self.timestamp = timestamp
        self.data = data
        self.previous_hash = previous_hash
        self.hash = self.calculate_hash()

    def calculate_hash(self):
        value = str(self.index) + str(self.timestamp) + str(self.data) + str(self.previous_hash)
        return hashlib.sha256(value.encode()).hexdigest()

class Blockchain:
    def __init__(self):
        self.chain = [self.create_genesis_block()]

    def create_genesis_block(self):
        return Block(0, time.time(), "🚀 Genesis AIVT v9", "0")

    def add_block(self, data):
        previous = self.chain[-1]
        new_block = Block(len(self.chain), time.time(), data, previous.hash)
        self.chain.append(new_block)

    def is_valid(self):
        for i in range(1, len(self.chain)):
            curr = self.chain[i]
            prev = self.chain[i - 1]
            if curr.hash != curr.calculate_hash():
                return False
            if curr.previous_hash != prev.hash:
                return False
        return True

    def export_chain(self):
        return json.dumps([block.__dict__ for block in self.chain], indent=2, ensure_ascii=False)

    def save_to_file(self, filename="aivt_chain.json"):
        with open(filename, "w", encoding="utf-8") as f:
            f.write(self.export_chain())

def run_app():
    chain = Blockchain()
    print("✅ Khởi động AIVT Blockchain v9")
    while True:
        print("\\nMenu:")
        print("1. Thêm khối mới")
        print("2. Xuất chuỗi")
        print("3. Kiểm tra chuỗi")
        print("4. Lưu vào file")
        print("0. Thoát")
        choice = input("Lựa chọn: ")
        if choice == "1":
            data = input("Nhập dữ liệu khối: ")
            chain.add_block(data)
            print("✅ Đã thêm khối")
        elif choice == "2":
            print("📦 Chuỗi khối AIVT:")
            print(chain.export_chain())
        elif choice == "3":
            print("✅ Chuỗi hợp lệ:", chain.is_valid())
        elif choice == "4":
            chain.save_to_file()
            print("💾 Đã lưu vào aivt_chain.json")
        elif choice == "0":
            break
        else:
            print("❌ Lựa chọn không hợp lệ")

if __name__ == "__main__":
    run_app()
"""

# Lưu file script
script_path = os.path.join(project_dir, "aivt_blockchain_app.py")
with open(script_path, "w", encoding="utf-8") as f:
    f.write(main_script)

# Tạo file README
readme = """
# AIVT Blockchain v9 App (Console)

✅ Chạy trên Android qua Pydroid 3 hoặc Termux

## Cách chạy trên Android:
1. Tải Pydroid 3 từ Play Store.
2. Mở Pydroid 3 -> Open file `aivt_blockchain_app.py`
3. Bấm Run ▶️ để sử dụng.

Tính năng:
- Thêm khối mới với dữ liệu tùy ý
- Kiểm tra chuỗi hợp lệ
- Xuất chuỗi JSON
- Lưu vào file aivt_chain.json
"""
readme_path = os.path.join(project_dir, "README.txt")
with open(readme_path, "w", encoding="utf-8") as f:
    f.write(readme)

# Đóng gói zip
zip_path = "/mnt/data/AIVT_Blockchain_App.zip"
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
    for root, dirs, files in os.walk(project_dir):
        for file in files:
            full_path = os.path.join(root, file)
            relative_path = os.path.relpath(full_path, project_dir)
            zipf.write(full_path, arcname=relative_path)

zip_path