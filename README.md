## Tạo và kích hoạt môi trường ảo venv
Mở thư mục gốc (Thư mục có chứa thư mục src)
```bash
py -m venv .venv
```
Window: 
```bash
.venv\Scripts\activate
```
macOS / Linux: 
```bash
source .venv/bin/activate
```
## Cài đặt các thư viện cần thiết
```bash
pip install -r requirements.txt
```
## Hướng dẫn chạy chương trình
1.  **Chuẩn bị:**
    Tải và mở thư mục gốc của dự án
2.  **Chạy chương trình chính:**
    Sử dụng lệnh sau để thực thi chương trình:

    ```bash
    python src/main.py
    ```
3.  **Kết quả:**
    Chương trình sẽ thực hiện lần lượt:
    * Phân tích (Parse) file PNML và kiểm tra tính nhất quán.
    * Tìm không gian trạng thái bằng thuật toán BFS (Task 2).
    * Tìm không gian trạng thái bằng thuật toán BDD (Task 3).
    * So sánh kết quả (số lượng trạng thái) và thời gian và bộ nhớ chạy giữa hai phương pháp.
    * Tìm Deadlock của mạng PetriNet bằng cách kết hợp BDD và ILP (Task 4).
    * Tối ưu hóa hàm mục tiêu trên tập trạng thái đạt tới bằng cách kết hợp BDD và ILP (Task 5).
    * Báo cáo thời gian chạy cho việc tìm deadlock và tối ưu hóa hàm mục tiêu trên các mô hình mẫu.
# 📂 Cấu trúc Dự án

Mã nguồn được chia thành các module biệt lập để dễ quản lý và bảo trì:

* `main.py`: Điểm khởi chạy của chương trình. Điều phối việc đọc file, gọi các thuật toán và so sánh kết quả.
* `petrinet_model.py`: Định nghĩa các lớp dữ liệu (`Place`, `Transition`, `Arc`, `PetriNet`) và chứa logic kiểm tra tính nhất quán (`verify_consistency`).
* `parse_pnml.py`: Chứa hàm xử lý đọc file XML (`.pnml`) và chuyển đổi thành đối tượng `PetriNet`.
* `find_reachable_byBFS.py`: Cài đặt thuật toán tìm kiếm theo chiều rộng (BFS) để tính toán không gian trạng thái một cách tường minh (Task 2).
* `find_reachable_byBDD.py`: Cài đặt thuật toán tính toán tượng trưng sử dụng Binary Decision Diagrams (BDD) với thư viện `pyeda` (Task 3).
* `file.pnml`: File dữ liệu đầu vào mẫu (Mạng Petri).
* `generate_pnml.py`: File tạo sinh ra Mạng Petri ngẫu nhiên và lưu vào trong folder `/testcase` với chỉ mục tự động
* `deadlock_detection_by_ILP_BDD.py` : Cài đặt logic phát hiện `Deadlock` bằng cách sử dụng kết hợp `BDD` và `ILP` (Task 4). 
* `optimization.py` : Thuật toán tối ưu hóa hàm mục tiêu trên tập trạng thái đạt được, kết hợp `BDD` và `ILP` cùng phương pháp `Cut Generation` (Task 5).
# ✨ Các tính năng đã thực hiện

* **Task 1: PNML Parsing**
    * Đọc file chuẩn PNML.
    * Xây dựng biểu diễn nội bộ (Internal Representation).
    * Kiểm tra tính nhất quán (Consistency Check) của mạng.
* **Task 2: Explicit Reachability (BFS)**
    * Duyệt đồ thị trạng thái bằng hàng đợi (Queue).
    * Lưu trữ trạng thái bằng `frozenset`.
* **Task 3: Symbolic Reachability (BDD)**
    * Mã hóa trạng thái và chuyển tiếp bằng biến Boolean.
    * Sử dụng thư viện `pyeda` để tính toán điểm bất động (Fixed-point iteration).
    * Tối ưu hóa tính toán trên không gian trạng thái lớn.
* **Task 4: Deadlock Detection (BDD + ILP)**
    * Sử dụng BDD để biểu diễn tập hợp các Marking $M_{dead}$ trong đó tất cả các Transition đều bị Disable (Dead Condition).
    * Lấy giao của tập trạng thái Dead và tập Reachable đã tính ở Task 3: $C_{dead} = M_{dead} \cap Reach(M_0)$ làm tập ứng viên.
    * Sử dụng Gurobi để chọn Marking tối ưu nhất (ví dụ: tối đa hóa tổng số token/weight) từ các ứng cử viên $C_{dead}$.
* **Task 5: Optimization Over Reachable Marking (BDD + ILP + Cut Generation)**
    * `Mô hình ILP cơ bản`: Xây dựng mô hình tối ưu hóa ILP (sử dụng Gurobi) để tối đa hóa hàm mục tiêu tuyến tính $c^T M$.
    * `Ràng buộc Phương trình Trạng thái`: Thêm ràng buộc $M = M_0 + C \cdot Y$ (State Equations) vào mô hình ILP.
    * `Sử dụng Cut Generation`: Thực hiện vòng lặp Cut Generation (sinh ra ràng buộc) để kiểm tra tính Reachability của giải pháp tối ưu $M^*$ bằng BDD. Nếu không đạt được, thêm một cắt (cut) vào ILP để loại bỏ nó và lặp lại.
## 📘 Module: `petrinet_model.py`

Module này đóng vai trò là **xương sống (backbone)** của toàn bộ dự án. Nó định nghĩa các cấu trúc dữ liệu cốt lõi để biểu diễn một mạng Petri trong bộ nhớ máy tính.

Thay vì chỉ lưu trữ dữ liệu thô, module này được thiết kế tối ưu để hỗ trợ các thuật toán tìm kiếm (BFS, BDD) chạy hiệu quả nhất.

### 1. Các Lớp Cơ Bản (Basic Classes)
Đây là các thành phần nhỏ nhất cấu thành nên mạng:

* **`class Place`**: Đại diện cho một Vị trí.
    * Chứa `id` (định danh duy nhất).
    * Phương thức `__repr__` được ghi đè để hiển thị khi debug (ví dụ: `Place(id='p1')`).
* **`class Transition`**: Đại diện cho một Chuyển tiếp.
* **`class Arc`**: Đại diện cho một Cung nối, chứa thông tin `source_id` (nguồn) và `target_id` (đích).

### 2. Lớp Chính: `class PetriNet`
Lớp hoạt động như một "thùng chứa" (container) cho toàn bộ mạng lưới.

#### Cấu trúc dữ liệu (`__init__`)
Module duy trì hai hệ thống lưu trữ song song để tối ưu hóa hiệu năng truy xuất:

```python
# Tra cứu nhanh Place theo ID
self.places: Dict[str, Place] = {}   

# Tra cứu nhanh Transition theo ID
self.transitions: Dict[str, Transition] = {}

# Danh sách tuần tự (dùng cho in ấn/kiểm tra), không cần tra cứu bằng ID
self.arcs: list[Arc] = []               

# Tập hợp các Place có token ban đầu
self.initial_marking: Set[str] = set()   

# CẤU TRÚC TỐI ƯU CHO DUYỆT ĐỒ THỊ (O(1)):
self.pre: Dict[str, Set[str]] = {}       # Tập hợp đầu vào của mỗi Transition
self.post: Dict[str, Set[str]] = {}      # Tập hợp đầu ra của mỗi Transition
```
Lý do thiết kế:
* `self.pre` và `self.post` đóng vai trò như bảng tra cứu tốc độ cao. Thuật toán BFS/BDD có thể lấy ngay lập tức danh sách các Place cần thiết để kích hoạt một Transition mà không cần duyệt lại toàn bộ danh sách cung, giúp giảm độ phức tạp thuật toán.
* Sử dụng *Dictionary (Hash Map)* cho `places` và `transitions` để tối ưu hóa việc truy xuất theo ID về độ phức tạp $O(1)$, giúp quá trình parsing file lớn không bị chậm.
* Sử dụng *Set (Hash Set)* cho `pre` và `post` conditions để tận dụng các phép toán tập hợp mạnh mẽ của Python (`issubset`, `union`, `difference`), yếu tố cốt lõi giúp thuật toán BFS và BDD đạt hiệu suất cao.

#### Quản lý Chuyển tiếp (`add_transition`)
Hàm này thực hiện nhiều việc hơn là chỉ lưu trữ dữ liệu. Nó đóng vai trò **khởi tạo cấu trúc đồ thị**.

```python
def add_transition(self, trans: Transition):
    self.transitions[trans.id] = trans
    # KHỞI TẠO CẤU TRÚC ĐỒ THỊ:
    # Tạo trước các tập hợp rỗng cho đầu vào (pre) và đầu ra (post)
    self.pre[trans.id] = set()
    self.post[trans.id] = set()
```

#### Xử lý Logic Cung (`add_arc`)
Hàm chuyển đổi danh sách cung tuyến tính thành logic vận hành của mạng khi đọc file.
```python
def add_arc(self, arc: Arc):
    self.arcs.append(arc) # Lưu vào danh sách thô (để in ấn/debug)

    # PHÂN LOẠI VÀ XÂY DỰNG LOGIC:
    
    # 1. Cung đi từ Place -> Transition (Điều kiện kích hoạt - Pre)
    if arc.source_id in self.places and arc.target_id in self.transitions:
        self.pre[arc.target_id].add(arc.source_id)

    # 2. Cung đi từ Transition -> Place (Hệ quả kích hoạt - Post)
    elif arc.source_id in self.transitions and arc.target_id in self.places:
        self.post[arc.source_id].add(arc.target_id)
```
* **Cơ chế hoạt động:** Thay vì để các thuật toán (BFS/BDD) phải tự đi tìm xem "cung này nối cái gì với cái gì" mỗi lần chạy, hàm này phân loại ngay lập tức:
    * `self.pre[t]`: Chứa tất cả các token cần phải thu hồi khi `Transition` khai hỏa.
    * `self.post[t]`: Chứa tất cả các token sẽ được sinh ra khi `Transition` khai hỏa.

#### Thiết lập Trạng thái Đầu (`set_initial_marking`)
Hàm thiết lập điểm xuất phát ($M_0$) cho toàn bộ quá trình phân tích.
```python
def set_initial_marking(self, place_id: str):
    # Kiểm tra an toàn: Chỉ thêm nếu Place thực sự tồn tại
    if place_id in self.places:
        self.initial_marking.add(place_id)
    else:
        print(f"Cảnh báo: Đánh dấu ban đầu cho Vị trí không tồn tại '{place_id}'")
```
* **Tính chất**: Sử dụng `Set` để lưu trữ đảm bảo tính duy nhất. Trong mạng 1-safe, dù file input có lỡ khai báo trùng lặp (<initialMarking> xuất hiện 2 lần cho cùng 1 chỗ), logic chương trình vẫn đúng (chỉ lưu 1 lần).

#### Kiểm tra Nhất quán (`verify_consistency`)
Hàm này đóng vai trò là chốt chặn an toàn (Sanity Check) cho **Task 1**. Nó duyệt qua toàn bộ danh sách cung đã lưu trữ để kiểm tra tính toàn vẹn của cấu trúc mạng.

```python
def verify_consistency(self):
    # Tạo một tập hợp chứa tất cả các ID hợp lệ (Places + Transitions)
    all_nodes = self.places.keys() | self.transitions.keys()

    errors_found = False
    for arc in self.arcs:
        # Kiểm tra nguồn cung
        if arc.source_id not in all_nodes:
            print(f"Lỗi: Nguồn '{arc.source_id}' không tồn tại.")
            errors_found = True
            
        # Kiểm tra đích cung
        if arc.target_id not in all_nodes:
            print(f"Lỗi: Đích '{arc.target_id}' không tồn tại.")
            errors_found = True
            
    return not errors_found
```
Mục đích:
* Đảm bảo rằng mọi cung đều kết nối với các Nút (Node) thực sự tồn tại trong mạng.
* Ngăn chặn các lỗi "dangling pointer" (con trỏ treo) hoặc lỗi runtime khi chạy thuật toán duyệt đồ thị sau này.

## 📗 Module: `parse_pnml.py` 

Module này thực hiện **Task 1** của dự án, chuyển đổi dữ liệu thô từ file XML (`.pnml`) thành các đối tượng Python (`PetriNet`) mà chương trình có thể hiểu và xử lý được.

### Các thành phần chính

#### Thư viện sử dụng
* `xml.etree.ElementTree`: Thư viện chuẩn của Python dùng để phân tích cú pháp XML. Được chọn vì tính đơn giản, hiệu quả và không yêu cầu cài đặt thêm (built-in).

#### Hàm chính: `parse_pnml(file_path)`

Quy trình xử lý dữ liệu diễn ra theo các bước tuần tự sau:

1.  **Khởi tạo:**
    * Tạo một đối tượng `PetriNet` rỗng để chứa dữ liệu.
    * Sử dụng `ET.parse(file_path)` để đọc file và tạo cây XML (ElementTree).

2.  **Duyệt cây XML (Parsing Loop):**
    Hàm sử dụng phương thức `.findall()` với XPath (`.//tag`) để tìm kiếm các thẻ cần thiết bất kể chúng nằm sâu bao nhiêu trong cấu trúc file.

    * **Bước 2a - Đọc Vị trí (Places):**
        * Tìm tất cả thẻ `<place>`.
        * Lấy thuộc tính `id`.
        * Kiểm tra thẻ con `<initialMarking>` để xác định trạng thái đầu (token).
        * Gọi `net.add_place()` và `net.set_initial_marking()`.

    * **Bước 2b - Đọc Chuyển tiếp (Transitions):**
        * Tìm tất cả thẻ `<transition>`.
        * Lấy thuộc tính `id`.
        * Gọi `net.add_transition()`. *Lưu ý: Bước này cũng tự động khởi tạo cấu trúc đồ thị `pre`/`post` rỗng.*

    * **Bước 2c - Đọc Cung (Arcs):**
        * Tìm tất cả thẻ `<arc>`.
        * Lấy các thuộc tính `id`, `source`, `target`.
        * Gọi `net.add_arc()`. *Lưu ý: Bước này kích hoạt logic "biên dịch" để điền dữ liệu vào cấu trúc `pre`/`post`.*

3.  **Kiểm tra Nhất quán (Validation):**
    * Sau khi đọc xong, gọi `net.verify_consistency()` để đảm bảo dữ liệu toàn vẹn.
    * Nếu phát hiện lỗi (ví dụ: cung trỏ đến ID không tồn tại), hàm sẽ trả về `None` để ngăn chương trình chạy tiếp với dữ liệu sai.

4.  **Xử lý Lỗi (Error Handling):**
    * `ET.ParseError`: Bắt lỗi nếu file XML bị hỏng (cú pháp sai).
    * `FileNotFoundError`: Bắt lỗi nếu đường dẫn file không đúng.

## 📙 Module: `find_reachable_byBFS.py`

Module này thực hiện **Task 2** của dự án: Tính toán tường minh (Explicit) không gian trạng thái của mạng Petri bằng thuật toán **Breadth-First Search (BFS)**.

### Mục tiêu
Liệt kê toàn bộ các trạng thái (markings) có thể tới được từ trạng thái ban đầu bằng cách duyệt qua từng trạng thái một.

### Thiết kế Giải thuật

Thuật toán sử dụng mô hình "vết dầu loang" để khám phá không gian trạng thái.

#### 1. Khởi tạo
* Sử dụng `deque` (Double-ended Queue) từ thư viện `collections` để làm hàng đợi (queue) cho BFS. 
* Sử dụng `Set` để lưu trữ danh sách các trạng thái đã thăm (`visited`) tránh lặp vô hạn và giảm thời gian kiểm tra trùng lặp xuống $O(1)$.
* **Quan trọng:** Các trạng thái được lưu dưới dạng `frozenset`.
    * `set` trong Python là mutable (có thể thay đổi) nên không thể hash được và không thể thêm vào một `set` khác.
    * `frozenset` là immutable (bất biến), cho phép hash và lưu trữ an toàn trong tập `visited`.

#### 2. Vòng lặp BFS (BFS Loop)
Quá trình duyệt diễn ra như sau:

```python
while q:
    # 1. Lấy trạng thái đầu tiên ra khỏi hàng đợi
    current_marking = q.popleft()

    # 2. Duyệt qua TẤT CẢ các transition trong mạng
    for trans_id in net.transitions.keys():
        
        # 3. KIỂM TRA KÍCH HOẠT (Enabled Check)
        # Lấy tập hợp các Place đầu vào (Pre-set) của transition
        input_places = net.pre[trans_id]
        
        # Kiểm tra xem current_marking có chứa đủ các token đầu vào không
        # Sử dụng issubset() của Set (được tối ưu hóa cao trong CPython)
        if input_places.issubset(current_marking):
            
            # 4. KHAI HỎA (Fire) & TÍNH TRẠNG THÁI MỚI
            # Công thức: New = (Current - Pre) U Post
            output_places = net.post[trans_id]
            new_state = frozenset(current_marking - input_places | output_places)
            
            # 5. KIỂM TRA ĐÃ THĂM
            if new_state not in visited:
                q.append(new_state)
                visited.add(new_state)
```

## 📕 Module: `find_reachable_byBDD.py`

Module này thực hiện **Task 3: Tính toán Tượng trưng (Symbolic Reachability)**. Thay vì duyệt từng trạng thái riêng lẻ, thuật toán sử dụng **Binary Decision Diagrams (BDD)** để thao tác trên các tập hợp trạng thái khổng lồ cùng một lúc.


```python
import time
from functools import reduce
from dd.autoref import BDD # Sử dụng thư viện 'dd' (Pure Python)
```


### HÀM HỖ TRỢ: LOGIC CÂN BẰNG (BALANCED LOGIC)
`def balanced_op_iterative(manager, bdd_list, op_type="and")`

Thực hiện phép AND/OR trên danh sách BDD lớn.

- Tránh cây nghiêng (Skewed Tree): Vòng lặp tuần tự tạo ra cây biểu thức 
    lệch, không tối ưu cho BDD.
- Tránh lỗi đệ quy (RecursionError): Với hàng nghìn node (mạng lớn), 
    đệ quy thông thường sẽ gây tràn ngăn xếp Python.

- Sử dụng Hàng đợi (Queue) để gom cặp dần dần (Pairwise Merging).
- Đảm bảo độ sâu cây tính toán chỉ là O(log N).

### THUẬT TOÁN CHÍNH
```python
def find_reachable_byBDD(net: PetriNet):
    # A. KHỞI TẠO & CẤU HÌNH
    manager = BDD()
    
    # Tắt tính năng Sắp xếp động (Dynamic Reordering).
    # Lý do: Việc thư viện tự động dừng tính toán để sắp xếp lại biến tốn 
    # rất nhiều CPU. Ta sẽ chủ động sắp xếp tối ưu ngay từ đầu.
    manager.configure(reordering=False)

    # B. MÃ HÓA BIẾN (VARIABLE ENCODING)
    # Chiến lược: Interleaving Variable Ordering (Sắp xếp xen kẽ)
    # Thay vì khai báo hết v (v1..vn) rồi đến v' (v'1..v'n), ta khai báo:
    # v1, v'1, v2, v'2...
    # Lý do: Mối quan hệ giữ nguyên trạng thái (v == v') là phổ biến nhất.
    # Đặt chúng cạnh nhau giúp giảm kích thước BDD từ hàm mũ xuống tuyến tính.
    for place_id in sorted(net.places.keys()):
        p_prime = place_id + "_prime"
        manager.declare(place_id)
        manager.declare(p_prime)
        # ...

    # C. XÂY DỰNG QUAN HỆ CHUYỂN TIẾP (PARTITIONED TRANSITION RELATION)
    # Thay vì gộp tất cả thành một Monolithic TR khổng lồ (dễ tràn RAM),
    # ta xây dựng luật riêng cho từng Transition t:
    # TR_t = Pre_t(v) AND Post_t(v') AND Frame_t(v, v')
    
    # Tối ưu Frame Condition:
    # Tính trước bản đồ tương đương (Equivalence Map): (v <-> v')
    # Giúp tái sử dụng node, tránh tạo lặp lại hàng nghìn biểu thức giống nhau.
    equiv_map = {} 
    for p in place_ids:
        equiv_map[p] = (v[p] & v_prime[p]) | (~v[p] & ~v_prime[p])

    tr_list = []
    for trans_id in net.transitions:
        # ...
        # Pre-condition: Đầu vào phải có token
        pre_cond = manager.true
        for p in pre:
            pre_cond &= v[p]
        
        # Post-condition: Đầu ra nhận token, đầu vào mất token
        post_cond = manager.true
        for p in post:
            post_cond &= v_prime[p]  # p' = 1
        for p in pre_only:
            post_cond &= ~v_prime[p]  # p' = 0
        
        # Frame-condition: Các nơi KHÔNG liên quan phải giữ nguyên giá trị
        # Sử dụng Balanced AND để gộp hàng trăm điều kiện này lại
        frame_cond = bdd_and(manager, [equiv_map[p] for p in unchanged])
        
        tr_list.append(pre_cond & post_cond & frame_cond)

    # Gộp tất cả lại (Disjunction)
    TR = bdd_or(manager, tr_list)

    # D. VÒNG LẶP TÌM ĐIỂM BẤT ĐỘNG (FRONTIER SET FIXPOINT)
    # Chiến lược: Chỉ xử lý những trạng thái "Mới tìm thấy" (Frontier)
    
    Reachable = initial_state
    New_States = initial_state # Tập biên (Frontier)

    while True:
        # Bước 1: Tính ảnh (Image Computation)
        # Lọc ra các bước đi hợp lệ từ tập Frontier
        step_rel = New_States & TR
        if step_rel == manager.false: break 

        # Lượng từ tồn tại (Existential Quantification): 
        # Loại bỏ biến hiện tại (v), chỉ giữ lại biến tương lai (v')
        # (Đây là bước tốn kém nhất của thuật toán)
        current_step_image_prime = manager.exist(all_v_vars, step_rel)

        # Bước 2: Đổi tên biến (Renaming) v' -> v
        # Để chuẩn bị làm đầu vào cho vòng lặp kế tiếp
        New_v = manager.let(rename_map, current_step_image_prime)

        # Bước 3: Tính tập "Thực sự mới" (Difference)
        # Chỉ giữ lại những trạng thái chưa từng có trong Reachable
        # Phép toán này nhanh (O(1)) hơn so với so sánh Reachable == OldReachable
        really_new_states = New_v & ~Reachable
        
        if really_new_states == manager.false: 
            break # Dừng nếu không tìm thấy gì mới

        # Bước 4: Cập nhật
        Reachable = Reachable | really_new_states
        New_States = really_new_states # Cập nhật Frontier

    return Reachable
```
## 💻 Module: `deadlock_detection_by_ILP_BDD.py` (Task 4)

Module này thực hiện **Phát hiện Deadlock** bằng phương pháp tích hợp **BDD** (Symbolic Reachability) và **Integer Linear Programming (ILP)** (Tối ưu hóa lựa chọn).

### Mục tiêu
Tìm kiếm một *Deadlock Marking* ($M_{dead}$) **duy nhất** có thể đạt được từ trạng thái ban đầu ($M_{dead} \in$ Reach($M_0$)) và tối ưu hóa việc chọn Marking có trọng số (hoặc số token) cao nhất. 

### 1. Xây dựng Điều kiện Deadlock (BDD Logic)

Thuật toán sử dụng BDD để xác định tập hợp các trạng thái Dead và giao với tập Reachable BDD (Reach($M_0$)).

* **Logic:** Một Marking $M$ là *dead* nếu **tất cả** các Transition $t$ đều bị Disable.
* **Mã hóa:** Với mạng 1-safe, $t$ bị Disable nếu có **ít nhất một** Place đầu vào ($\bullet t$) không có token (tức là $\bigvee_{p \in \bullet t} \neg M(p)$).
* **Xác định Ứng cử viên (Candidates):** Tính giao của tập trạng thái Dead ($M_{dead}$) và tập Reachable ($Reach(M_0)$) từ Task 3 để loại bỏ các Marking chết giả:
    ```python
    deadlock_candidates_bdd = reachable_bdd & is_dead_bdd
    ```

### 2. Lựa chọn Marking tối ưu (ILP Optimization)

#### Trích xuất Ứng cử viên
* Module duyệt qua `deadlock_candidates_bdd` và sử dụng hàm `manager.pick()` để trích xuất một số lượng giới hạn các Marking thực tế ($M \in C_{dead}$) để chuyển sang ILP.

#### Mô hình ILP
* Sử dụng Gurobi (ILP) để tạo mô hình lựa chọn, nơi mỗi Marking ứng cử viên là một biến nhị phân (`select_marking_i`).
* **Ràng buộc:** Bắt buộc **chỉ chọn 1 Marking** (`Select_One_Marking`).
* **Hàm Mục tiêu:** Tối đa hóa tổng trọng số (hoặc tổng số token) của Marking được chọn.

```python
# Ràng buộc: Chỉ chọn 1 marking
model.addConstr(gp.quicksum(marking_vars.values()) == 1, "Select_One_Marking")
# Mục tiêu: tối đa hóa số token/weight
model.setObjective(gp.quicksum(objective_terms), GRB.MAXIMIZE)
