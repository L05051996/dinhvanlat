import zipfile
import os

# Tạo lại thư mục chứa mã nguồn (do trạng thái kernel đã reset)
project_dir = "/mnt/data/AIVT_v9_Termux"
os.makedirs(project_dir, exist_ok=True)

# Mã nguồn Python AIVT v9 nâng cấp
python_code = '''
import hashlib, time, json, os

class Block:
    def __init__(self, index, timestamp, data, previous_hash):
        self.index = index
        self.timestamp = timestamp
        self.data = data
        self.previous_hash = previous_hash
        self.hash = self.calculate_hash()

    def calculate_hash(self):
        value = f"{self.index}{self.timestamp}{self.data}{self.previous_hash}"
        return hashlib.sha256(value.encode()).hexdigest()

class Blockchain:
    def __init__(self, chain_file="aivt_chain.json"):
        self.chain_file = chain_file
        self.chain = self.load_chain() if os.path.exists(chain_file) else [self.create_genesis_block()]
        self.save_chain()

    def create_genesis_block(self):
        return Block(0, time.time(), "🚀 Genesis Block AIVT v9", "0")

    def add_block(self, data):
        last = self.chain[-1]
        new_block = Block(len(self.chain), time.time(), data, last.hash)
        self.chain.append(new_block)
        self.save_chain()

    def is_valid(self):
        for i in range(1, len(self.chain)):
            if self.chain[i].hash != self.chain[i].calculate_hash():
                return False
            if self.chain[i].previous_hash != self.chain[i - 1].hash:
                return False
        return True

    def save_chain(self):
        with open(self.chain_file, "w", encoding="utf-8") as f:
            json.dump([b.__dict__ for b in self.chain], f, indent=2, ensure_ascii=False)

    def load_chain(self):
        with open(self.chain_file, "r", encoding="utf-8") as f:
            data = json.load(f)
            chain = []
            for block in data:
                blk = Block(block["index"], block["timestamp"], block["data"], block["previous_hash"])
                blk.hash = block["hash"]
                chain.append(blk)
            return chain

    def print_chain(self):
        print(json.dumps([b.__dict__ for b in self.chain], indent=2, ensure_ascii=False))

# Khởi tạo
aivt = Blockchain()
aivt.add_block("✅ Khối 1: Tri thức")
aivt.add_block("🧠 Khối 2: Tự học")
aivt.add_block("⚙️ Khối 3: Tự nâng cấp")
aivt.add_block("🌍 Khối 4: Đồng bộ toàn cầu")
aivt.add_block("🔐 Khối 5: Bảo vệ lượng tử")
aivt.print_chain()
print("Chuỗi hợp lệ:", aivt.is_valid())
'''

# Ghi vào file
file_path = os.path.join(project_dir, "AIVT_v9.py")
with open(file_path, "w", encoding="utf-8") as f:
    f.write(python_code)

# Đóng gói thành file zip
zip_path = "/mnt/data/AIVT_v9_Termux.zip"
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
    for root, dirs, files in os.walk(project_dir):
        for file in files:
            full_path = os.path.join(root, file)
            relative_path = os.path.relpath(full_path, project_dir)
            zipf.write(full_path, arcname=relative_path)

zip_path