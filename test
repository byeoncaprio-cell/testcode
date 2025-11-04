import tkinter as tk
from tkinter import ttk, messagebox, filedialog, simpledialog
from typing import Set, Tuple, List, Dict, Union, Optional
from dataclasses import dataclass, field
from collections import Counter, OrderedDict
import random
import copy

# openpyxl은 Import 기능에 필요합니다.
from openpyxl import load_workbook

# --------------------------------
# Types & Models
# --------------------------------
Cell = Tuple[int, int]  # (row, col) 1-based

@dataclass
class Block:
    rows: int
    cols: int
    hatch: str = ""
    hold: str = ""
    bay: str = ""
    deck: str = ""
    cell_colors: Dict[Cell, str] = field(default_factory=dict)
    cell_numbers: Dict[Cell, Union[int, float, str]] = field(default_factory=dict)
    sockets: Set[Cell] = field(default_factory=set)
    gang_counts: Dict[int, int] = field(default_factory=dict)
    is_hold: bool = field(init=False)

    def __post_init__(self):
        self.is_hold = self.rows >= 6

@dataclass
class SectionHeader:
    title: str

Item = Union[Block, SectionHeader]


# --------------------------------
# Shape Library for Auto-Set
# --------------------------------
SHAPE_LIBRARY: Dict[int, List[List[Cell]]] = {
: [
        [(0, 0), (0, 1), (0, 2), (1, 0), (1, 1), (1, 2), (2, 0), (2, 1), (2, 2)],
        [(0, 0), (0, 1), (0, 2), (0, 3), (0, 4), (1, 0), (1, 1), (1, 2), (1, 3)],
        [(0, 0), (0, 1), (0, 2), (0, 3), (1, 0), (1, 1), (1, 2), (1, 3), (1, 4)],
        [(0, 1), (1, 1), (1, 2), (2, 0), (2, 1), (2, 2), (3, 0), (3, 1), (4, 1)],
        [(0, 0), (1, 0), (1, 1), (1, 2), (2, 0), (2, 1), (2, 2), (3, 0), (3, 1)],
        [(0, 1), (1, 0), (1, 1), (2, 0), (2, 1), (2, 2), (3, 0), (3, 1), (3, 2)],
        [(0, 1), (0, 2), (0, 3), (0, 4), (1, 0), (1, 1), (1, 2), (1, 3), (1, 4)],
        [(0, 0), (0, 1), (0, 2), (0, 3), (0, 4), (1, 1), (1, 2), (1, 3), (1, 4)],
    ],
    8: [ 
        [(0, 0), (0, 1), (0, 2), (0, 3), (1, 0), (1, 1), (1, 2), (1, 3)],
        [(0, 0), (0, 1), (1, 0), (1, 1), (2, 0), (2, 1), (3, 0), (3, 1)],
        [(0, 1), (0, 2), (1, 0), (1, 1), (1, 2), (2, 0), (2, 1), (2, 2)],  
        [(0, 0), (0, 1), (1, 0), (1, 1), (1, 2), (2, 0), (2, 1), (2, 2)],  
    ],
    7: [
        [(0, 0), (0, 1), (0, 2), (0, 3), (1, 0), (1, 1), (1, 2)],
        [(0, 0), (0, 1), (0, 2), (1, 0), (1, 1), (1, 2), (1, 3)],
        [(0, 1), (1, 0), (1, 1), (2, 0), (2, 1), (3, 0), (3, 1)],
        [(0, 0), (0, 1), (1, 0), (1, 1), (2, 0), (2, 1), (3, 1)],
        [(0, 0), (0, 1), (0, 2), (0, 3), (1, 1), (1, 2), (1, 3)],
        [(0, 1), (0, 2), (0, 3), (1, 0), (1, 1), (1, 2), (1, 3)],
        [(0, 0), (1, 0), (1, 1), (2, 0), (2, 1), (3, 0), (3, 1)],
        [(0, 0), (0, 1), (1, 0), (1, 1), (2, 0), (2, 1), (3, 0)],
        [(0, 0), (1, 0), (1, 1), (1, 2), (2, 0), (2, 1), (2, 2)],
        [(0, 2), (1, 0), (1, 1), (1, 2), (2, 0), (2, 1), (2, 2)],
    ],
    6: [
        [(0, 0), (0, 1), (0, 2), (1, 0), (1, 1), (1, 2)],  # 2x3
        [(0, 0), (0, 1), (1, 0), (1, 1), (2, 0), (2, 1)],  # 3x2
    ],
    5: [
        [(0, 0), (0, 1), (0, 2), (0, 3), (0, 4)],  # 1x5
        [(0, 0), (1, 0), (2, 0), (3, 0), (4, 0)],  # 5x1
    ],
    4: [
        [(0, 0), (0, 1), (1, 0), (1, 1)],  # 2x2
    ]
}

# --------------------------------
# Utils (필수 헬퍼 함수)
# (parse_number_like, _collect_rd_counts, _rd_list_for_rs, _build_rd_queues, _build_rs_summary 는 동일)
# --------------------------------
def parse_number_like(s: str) -> Union[int, float, str]:
    """Import에 필요"""
    try:
        if str(s).strip() == "": return ""
        if "." in str(s):
            f = float(s)
            return int(f) if f.is_integer() else f
        return int(s)
    except (ValueError, TypeError):
        return s

def _collect_rd_counts(items: List[Item]) -> OrderedDict:
    """Live Summary에 필요"""
    c = Counter(int(v) for it in items if isinstance(it,Block) for v in it.cell_numbers.values() if str(v).strip())
    return OrderedDict(sorted(c.items()))

def _rd_list_for_rs(rs_index, rd_per_rs):
    """Live Summary 및 Auto Set에 필요"""
    if rd_per_rs <= 0 or rs_index <= 0: return []
    if rs_index % 2 == 1:
        # STBD (Odd)
        block_idx = (rs_index - 1) // 2
        start_odd = 1 + 2 * (block_idx * rd_per_rs)
        return [start_odd + 2*i for i in range(rd_per_rs)]
    else:
        # PORT (Even)
        block_idx = (rs_index // 2) - 1
        start_even = 2 + 2 * (block_idx * rd_per_rs)
        return [start_even + 2*i for i in range(rd_per_rs)]

def _build_rd_queues(rs_total, rpr):
    """Auto Set에 필요"""
    even_q, odd_q = [], []
    for rs in range(1, rs_total + 1):
        (even_q if rs % 2 == 0 else odd_q).extend(_rd_list_for_rs(rs, rpr))
    return sorted(list(set(even_q))), sorted(list(set(odd_q)))

def _build_rs_summary(rs_indices, rd_counts, rpr):
    """Live Summary에 필요"""
    lines = []
    for rs in rs_indices:
        rd_list = _rd_list_for_rs(rs, rpr)
        total = sum(rd_counts.get(rd, 0) for rd in rd_list)
        lines.append(f"RS-{rs}: total {total}")
        lines.extend([f"  RD-{rd}: {rd_counts[rd]}" for rd in rd_list if rd_counts.get(rd, 0) > 0])
        if not any(rd_counts.get(rd, 0) > 0 for rd in rd_list):
            lines.append("  -")
        lines.append("")
    return "\n".join(lines).rstrip()

# --------------------------------
# 최소 기능 + 프리뷰 GUI
# --------------------------------
class MinimalAutoSetTester(ttk.Frame):
    def __init__(self, master):
        super().__init__(master, padding=10)
        self.pack(fill="both", expand=True)

        # --- 최소 상태 변수 ---
        self.FIXED_COLORS = ["#FFFF99", "#99CCFF", "#CCFFCC", "#F2DCDB"]
        self.rs_count = tk.StringVar(value="0")
        self.rd_per_rs = tk.StringVar(value="0")
        self.items: List[Item] = []
        
        self.rs_count.trace_add("write", self._recompute_all)
        self.rd_per_rs.trace_add("write", self._recompute_all)

        self._create_widgets()
        self._recompute_all()

    def _create_widgets(self):
        # 레이아웃: 우측(컨트롤), 좌측(리스트+프리뷰)
        outer = ttk.Frame(self); outer.pack(fill="both", expand=True)
        right = ttk.Frame(outer); right.pack(side="right", fill="y", padx=(10, 0), anchor="n")
        left = ttk.Frame(outer); left.pack(side="left", fill="both", expand=True)

        # --- 우측 (RS/RD 설정 및 요약) ---
        cfg = ttk.LabelFrame(right, text="RS / RD Panels", padding=10)
        cfg.pack(fill="x", pady=(0, 10))
        ttk.Label(cfg, text="RS Panels").grid(row=0, column=0, sticky="e", padx=5, pady=2)
        ttk.Spinbox(cfg, from_=0, to=999, textvariable=self.rs_count, width=8).grid(row=0, column=1, sticky="w", pady=2)
        ttk.Label(cfg, text="RD per RS").grid(row=0, column=2, sticky="e", padx=5, pady=2)
        ttk.Spinbox(cfg, from_=0, to=999, textvariable=self.rd_per_rs, width=8).grid(row=0, column=3, sticky="w", pady=2)

        summary_frame = ttk.LabelFrame(right, text="Live Summary", padding=10)
        summary_frame.pack(fill="both", expand=True, pady=(0, 10)) 

        self.grand_total_label = ttk.Label(summary_frame, text="Grand Total: 0", font=("Segoe UI", 9, "bold"))
        self.grand_total_label.pack(anchor="w", pady=(0, 5))

        alloc = ttk.Frame(summary_frame); alloc.pack(fill="both", expand=True)
        left_col = ttk.Frame(alloc); left_col.pack(side="left", fill="both", expand=True, padx=(0,5))
        ttk.Label(left_col, text="PORT RS").pack(anchor="w")
        self.txt_even = tk.Text(left_col, width=15, height=20, font=("Consolas", 10)); self.txt_even.pack(fill="both", expand=True); self.txt_even.configure(state="disabled")
        
        right_col = ttk.Frame(alloc); right_col.pack(side="left", fill="both", expand=True, padx=(5,0))
        ttk.Label(right_col, text="STBD RS").pack(anchor="w")
        self.txt_odd = tk.Text(right_col, width=15, height=20, font=("Consolas", 10)); self.txt_odd.pack(fill="both", expand=True); self.txt_odd.configure(state="disabled")

        # --- 좌측 (버튼, 리스트, 프리뷰) ---
        btn_row = ttk.Frame(left); btn_row.pack(fill="x", pady=5)
        ttk.Button(btn_row, text="Import…", command=self.import_from_excel).grid(row=0, column=0, padx=2, pady=2)
        ttk.Button(btn_row, text="Auto Set (Optimal)", command=self.auto_set_groups).grid(row=0, column=1, padx=(12,2), pady=2)

        preview_panel = ttk.Frame(left)
        preview_panel.pack(fill="both", expand=True, pady=5)
        
        list_frame = ttk.Frame(preview_panel)
        list_frame.pack(side="left", fill="both", expand=True, padx=(0, 5))
        ttk.Label(list_frame, text="Imported Items").pack(anchor="w")
        self.items_list = tk.Listbox(list_frame, height=20); self.items_list.pack(fill="both", expand=True)

        canvas_frame = ttk.Frame(preview_panel)
        canvas_frame.pack(side="left", fill="both", expand=True, padx=(5, 0))
        ttk.Label(canvas_frame, text="Layout Preview").pack(anchor="w")
        
        v_scroll = ttk.Scrollbar(canvas_frame, orient="vertical")
        h_scroll = ttk.Scrollbar(canvas_frame, orient="horizontal")
        
        self.canvas = tk.Canvas(canvas_frame, bg="white", highlightthickness=1, highlightbackground="#cccccc",
                                yscrollcommand=v_scroll.set,
                                xscrollcommand=h_scroll.set)
        
        v_scroll.config(command=self.canvas.yview)
        h_scroll.config(command=self.canvas.xview)
        
        v_scroll.pack(side="right", fill="y")
        h_scroll.pack(side="bottom", fill="x")
        self.canvas.pack(side="left", fill="both", expand=True)

    def draw_all_blocks_preview(self):
        """(프리뷰 기능) 캔버스에 모든 블럭을 순서대로 그립니다."""
        self.canvas.delete("all")
        
        cell_px = 10      # 프리뷰용 작은 셀 크기
        block_gap = 20    # 블럭 사이 세로 간격
        text_gap = 15     # 블럭 제목과 셀 그림 사이 간격
        current_y = 10    # 현재 Y 위치 (시작 패딩)
        left_padding = 10 # 좌측 패딩
        
        max_canvas_width = 0 # 스크롤바 영역 계산용

        for item in self.items:
            if isinstance(item, SectionHeader):
                self.canvas.create_text(
                    left_padding, current_y + text_gap/2,
                    text=f"--- {item.title} ---",
                    anchor="w", font=("Arial", 10, "bold")
                )
                current_y += block_gap
            
            elif isinstance(item, Block):
                b = item
                label = f"Hatch: {b.hatch}" if b.hatch else f"Hold: {b.hold}"
                
                self.canvas.create_text(
                    left_padding, current_y,
                    text=f"Block ({label}, {b.rows}x{b.cols})",
                    anchor="w", font=("Arial", 8)
                )
                current_y += text_gap 

                block_width = b.cols * cell_px
                max_canvas_width = max(max_canvas_width, left_padding + block_width)

                for r in range(1, b.rows + 1):
                    for c in range(1, b.cols + 1):
                        pos = (r, c)
                        if pos not in b.cell_colors:
                            continue 
                        
                        color = b.cell_colors.get(pos, "#FFFFFF")
                        number = b.cell_numbers.get(pos, "") 
                        
                        x1 = left_padding + (c - 1) * cell_px
                        y1 = current_y + (r - 1) * cell_px
                        x2 = x1 + cell_px
                        y2 = y1 + cell_px
                        
                        self.canvas.create_rectangle(x1, y1, x2, y2, fill=color, outline="#BBBBBB")
                        
                        if number:
                            self.canvas.create_text(
                                x1 + cell_px / 2, y1 + cell_px / 2,
                                text=str(number), font=("Arial", 6), 
                                fill="#000000"
                            )
                
                current_y += b.rows * cell_px + block_gap 
        
        self.canvas.config(scrollregion=(0, 0, max_canvas_width + 20, current_y))

    def _recompute_all(self, *args):
        """Live Summary 업데이트 트리거"""
        self._update_allocation_display()

    def _update_allocation_display(self):
        """Live Summary GUI를 실제 데이터로 업데이트"""
        try:
            rs_total = int(self.rs_count.get() or 0)
            rpr = int(self.rd_per_rs.get() or 0)
        except(ValueError, TypeError):
            rs_total = 0
            rpr = 0

        even_rs = [i for i in range(1, rs_total+1) if i % 2 == 0]
        odd_rs  = [i for i in range(1, rs_total+1) if i % 2 == 1]

        rd_counts = _collect_rd_counts(self.items)
        grand_total = sum(rd_counts.values())
        self.grand_total_label.config(text=f"Grand Total: {grand_total}")

        for txt_widget, indices in [(self.txt_even, even_rs), (self.txt_odd, odd_rs)]:
            content = _build_rs_summary(indices, rd_counts, rpr)
            txt_widget.config(state="normal")
            txt_widget.delete("1.0", tk.END)
            txt_widget.insert(tk.END, content)
            txt_widget.config(state="disabled")

    def refresh_list(self):
        """Item 리스트박스와 프리뷰 캔버스를 새로고침"""
        self.items_list.delete(0, tk.END)
        for i, it in enumerate(self.items, 1):
            self.items_list.insert(tk.END, self.item_label(it, i))
        
        self._recompute_all() 
        self.draw_all_blocks_preview() # 캔버스 프리뷰 업데이트

    def item_label(self, it, idx):
        """리스트박스에 표시될 아이템 이름 포맷"""
        if isinstance(it, SectionHeader): return f"[Section] {it.title}"
        b: Block = it
        label = f"Hatch:{b.hatch}" if b.hatch else (f"Hold:{b.hold}" if b.hold else "-")
        return f"Block {idx} — {b.rows}x{b.cols} | {label} | Nums:{len(b.cell_numbers)}"

    def import_from_excel(self):
        """엑셀 파일에서 블럭 레이아웃을 가져옴"""
        path = filedialog.askopenfilename(filetypes=[("Excel", "*.xlsx *.xlsm")])
        if not path: return
        try:
            wb = load_workbook(path, data_only=True)
            ws = wb.active
        except Exception as e:
            messagebox.showerror("Error", f"파일 열기 실패:\n{e}")
            return

        added, skipped = 0, 0
        default_fill_color = "#EEEEEE" 
        
        self.items.clear() 

        for row in ws.iter_rows(min_row=1, values_only=True):
            if not row or len(row) < 2 or row[0] is None or row[1] is None: continue
            try:
                values = [int(p.strip()) for p in str(row[1]).replace("，", ",").split(",") if p.strip()]
                if not values: raise ValueError
                rows, cols = len(values), max(values)
                
                b = Block(rows=rows, cols=cols)
                
                if rows >= 6:
                    b.hold = str(row[0]).strip()
                else:
                    b.hatch = str(row[0]).strip()
                    
                for r_idx, cnt in enumerate(values, 1):
                    start_c = 1 + (cols - cnt) // 2
                    for k in range(cnt): 
                        b.cell_colors[(r_idx, start_c + k)] = default_fill_color
                self.items.append(b)
                added += 1
            except Exception as e:
                print(f"Skipping row: {e}")
                skipped += 1
                
        self.refresh_list()
        messagebox.showinfo("Import", f"완료: {added}개 블럭 추가, {skipped}행 무시")

    # =================================================================
    # Auto Set 그룹 (메인 로직)
    # ⭐⭐⭐ 이 섹션의 코드를 수정하며 테스트하세요 ⭐⭐⭐
    # =================================================================
    def auto_set_groups(self):
        sel = self.items_list.curselection()
        targets = [self.items[sel[0]]] if sel and isinstance(self.items[sel[0]], Block) else [it for it in self.items if isinstance(it, Block)]
        if not targets:
            messagebox.showinfo("Auto Set", "대상 블럭이 없습니다.")
            return

        cap = simpledialog.askinteger("Auto Set", "RD panel 당 최대 컨테이너 수", minvalue=4, maxvalue=9999, parent=self)
        if cap is None: return 

        try:
            rs_total = int(self.rs_count.get() or 0)
            rpr = int(self.rd_per_rs.get() or 0)
        except(ValueError, TypeError):
            rs_total, rpr = 0, 0
            
        even_list, odd_list = _build_rd_queues(rs_total, rpr)

        # PASS 1: Tiling (공간을 조각으로 나누기)
        all_left_placements = []
        all_right_placements = []
        total_left_cells = 0
        total_right_cells = 0
        any_failure = False

        for b in targets:
            b.cell_numbers.clear() 
            b.gang_counts = {g: 0 for g in range(3, 10)}
            unfilled_cells = {p for p, color in b.cell_colors.items()}
            
            center_col = (b.cols + 1) / 2.0
            left_active = {p for p in unfilled_cells if p[1] < center_col}
            right_active = {p for p in unfilled_cells if p[1] > center_col}
            center_cells = {p for p in unfilled_cells if p[1] == center_col}

            if len(left_active) <= len(right_active):
                left_active.update(center_cells)
            else:
                right_active.update(center_cells)

            placements = [] 
            success = False
            
            # --- 로직 분기 ---
            if b.is_hold: 
                # (수정됨) HOLD: 1D 최적화 실행
                success = self._solve_line_tiling_optimal(b, left_active, "LEFT", placements) and \
                          self._solve_line_tiling_optimal(b, right_active, "RIGHT", placements)
            else:
                # (수정됨) HATCH: 2D 최적화(전체탐색) 실행
                # ⚠️ 이 부분이 매우 느릴 수 있습니다!
                print(f"Hatch {b.hatch} (2D Optimal) 탐색 시작...")
                right_placements, left_placements = [], []
                if self._solve_tiling_optimal_recursive(b, right_active, "RIGHT", right_placements):
                    if self._solve_tiling_optimal_recursive(b, left_active, "LEFT", left_placements):
                        placements = right_placements + left_placements
                        success = True
                print(f"Hatch {b.hatch} 탐색 완료.")

            if not success:
                messagebox.showerror("배치 실패", f"블록 {b.hatch or b.hold or '(번호 없음)'}에서 빈 칸 없이 모든 공간을 채우는 조합을 찾지 못했습니다.\n블록 모양을 확인해주세요.")
                any_failure = True
                continue

            # (이하는 원본과 동일)
            left_placements_b = [p for p in placements if p['side'] == 'LEFT']
            right_placements_b = [p for p in placements if p['side'] == 'RIGHT']

            left_placements_b.sort(key=lambda p: (min(r for r,c in p['cells']), min(c for r,c in p['cells'])))
            right_placements_b.sort(key=lambda p: (min(r for r,c in p['cells']), -max(c for r,c in p['cells'])))

            for p in left_placements_b:
                all_left_placements.append( (b, p) ) 
                total_left_cells += p['size']
                
            for p in right_placements_b:
                all_right_placements.append( (b, p) )
                total_right_cells += p['size']

        if any_failure:
            return

        # PASS 2: RD 번호 할당 (이 로직은 변경 없음)
        target_counts_even = {} 
        rd_remaining_even = {rd: cap for rd in even_list} 
        total_even_rds = len(even_list)
        if total_even_rds > 0 and total_left_cells > 0:
            avg_cap_even = total_left_cells // total_even_rds
            rem_even = total_left_cells % total_even_rds     
            for i, rd in enumerate(even_list):
                target_counts_even[rd] = avg_cap_even + (1 if i < rem_even else 0)
        
        target_counts_odd = {} 
        rd_remaining_odd = {rd: cap for rd in odd_list} 
        total_odd_rds = len(odd_list)
        if total_odd_rds > 0 and total_right_cells > 0:
            avg_cap_odd = total_right_cells // total_odd_rds 
            rem_odd = total_right_cells % total_odd_rds     
            for i, rd in enumerate(odd_list):
                target_counts_odd[rd] = avg_cap_odd + (1 if i < rem_odd else 0)

        cur_even = [0] 
        for (b, p) in all_left_placements:
            need = p['size'] 
            cells = p['cells'] 
            assigned_rd = self._rd_take_v10(need, target_counts_even, rd_remaining_even, even_list, cur_even)
            if assigned_rd is not None:
                color = self._pick_color(cells, b)
                for cell in cells:
                    b.cell_numbers[cell] = assigned_rd
                    b.cell_colors[cell] = color
                b.gang_counts[need] = b.gang_counts.get(need, 0) + 1

        cur_odd = [0] 
        for (b, p) in all_right_placements: 
            need = p['size'] 
            cells = p['cells'] 
            assigned_rd = self._rd_take_v10(need, target_counts_odd, rd_remaining_odd, odd_list, cur_odd)
            if assigned_rd is not None:
                color = self._pick_color(cells, b)
                for cell in cells:
                    b.cell_numbers[cell] = assigned_rd
                    b.cell_colors[cell] = color
                b.gang_counts[need] = b.gang_counts.get(need, 0) + 1

        self.refresh_list()
        self._recompute_all() 
        messagebox.showinfo("Auto Set", "자동 배치 완료")

    # =============================================================
    # ⭐ [수정된 로직 1] HOLD (1D) : DP로 최소 조각 찾기
    # =============================================================
    def _partition_optimal_dp(self, length: int, sizes: List[int], memo: Dict[int, Optional[List[int]]]) -> Optional[List[int]]:
        """
        (새로운 헬퍼 함수)
        다이나믹 프로그래밍(Memoization)을 사용해 'length'를 
        'sizes'의 조합으로 채우는 '최소 개수'의 조각 리스트를 반환합니다.
        """
        if length == 0:
            return []  # 0개 조각
        if length < 0:
            return None # 불가능
        if length in memo:
            return memo[length]

        best_solution = None
        min_pieces = float('inf')

        # [9, 8, 7, 6, 5, 4] 순서로 시도
        for size in sizes:
            if size <= length:
                res = self._partition_optimal_dp(length - size, sizes, memo)
                
                # 'res'가 유효한 해(None이 아님)이고
                if res is not None:
                    # 현재까지의 최고 기록(min_pieces)보다 더 적은 조각을 썼다면
                    if len(res) + 1 < min_pieces:
                        min_pieces = len(res) + 1
                        best_solution = [size] + res
        
        memo[length] = best_solution
        return best_solution

    def _solve_line_tiling_optimal(self, b: Block, side_active: Set[Cell], side: str, placements: List[Dict]) -> bool:
        """
        (수정된 HOLD 로직)
        각 행(row)을 순회하며, 연속된 셀(span)을 찾아 
        DP 헬퍼(_partition_optimal_dp)를 호출해 최적의(최소 개수) 조각으로 채웁니다.
        """
        # 최소 4개 이상인 조각들만 사용, 큰 순서대로
        valid_sizes = sorted([s for s in SHAPE_LIBRARY.keys() if s >= 4], reverse=True)

        rows = sorted(list({r for r, c in side_active}))
        for r in rows:
            cols_in_row = sorted([c for r_c, c in side_active if r_c == r])
            if not cols_in_row: continue
            
            spans = []
            start = cols_in_row[0]
            for i in range(1, len(cols_in_row)):
                if cols_in_row[i] != cols_in_row[i-1] + 1:
                    spans.append((start, cols_in_row[i-1]))
                    start = cols_in_row[i]
            spans.append((start, cols_in_row[-1]))
            
            for start_col, end_col in spans:
                span_len = end_col - start_col + 1
                
                # (수정) DP 함수 호출 (매번 새로운 memo 사용)
                pieces = self._partition_optimal_dp(span_len, valid_sizes, memo={})
                
                if pieces is None: 
                    return False # 이 줄은 도저히 못채움 -> 실패
                
                ptr = start_col
                for size in pieces:
                    cells = {(r, c) for c in range(ptr, ptr + size)}
                    placements.append({'size': size, 'cells': cells, 'side': side})
                    ptr += size
        return True

    # =============================================================
    # ⭐ [수정된 로직 2] HATCH (2D) : 전체 탐색으로 최소 조각 찾기
    # =============================================================
    def _solve_tiling_optimal_recursive(self, b: Block, unfilled: Set[Cell], side: str, placements: List[Dict]) -> bool:
        """
        (수정된 HATCH 로직 - 래퍼 함수)
        _find_all_solutions_helper를 호출하여 모든 가능한 해를 찾고,
        그 중 가장 조각 개수가 적은 해(best_solution)를 placements에 추가합니다.
        """
        all_solutions = [] # 찾은 모든 해를 저장할 리스트
        is_left = (side == "LEFT")
        
        # (최적화) 모든 2D/1D 모양을 미리 계산 (크기 내림차순 정렬)
        # 3x3(9) > 2x4(8) > ... > 2x2(4) > 1x9(9) > ... > 1x4(4)
        all_shapes = []
        priority_sizes = sorted(SHAPE_LIBRARY.keys(), reverse=True)
        
        # 1. 2D 모양 추가
        for size in priority_sizes:
            for shape_pattern in SHAPE_LIBRARY.get(size, []):
                shape_height = max(r_off for r_off, c_off in shape_pattern) + 1
                shape_width = max(c_off for r_off, c_off in shape_pattern) + 1
                if not (shape_height == 1 or shape_width == 1):
                    all_shapes.append({'size': size, 'pattern': shape_pattern})
        
        # 2. 1D 모양 추가
        for size in priority_sizes:
            for shape_pattern in SHAPE_LIBRARY.get(size, []):
                shape_height = max(r_off for r_off, c_off in shape_pattern) + 1
                shape_width = max(c_off for r_off, c_off in shape_pattern) + 1
                if shape_height == 1 or shape_width == 1:
                    all_shapes.append({'size': size, 'pattern': shape_pattern})

        # --- 실제 탐색을 수행할 헬퍼 함수 ---
        def _find_all_solutions_helper(current_unfilled: Set[Cell], current_placements: List[Dict]):
            
            if not current_unfilled:
                # 빈 칸을 모두 채웠으면, '해'로 인정하고 리스트에 추가
                all_solutions.append(copy.deepcopy(current_placements))
                return

            # (최적화) 가장 이른 셀을 찾아 이 셀을 덮는 경우만 탐색
            start_cell = min(current_unfilled, key=lambda p: (p[0], p[1] if is_left else -p[1]))

            # 모든 모양(2D, 1D)에 대해 시도
            for shape_info in all_shapes:
                pattern = shape_info['pattern']
                size = shape_info['size']
                
                # (최적화) 이 모양이 start_cell을 덮지 않으면 건너뜀
                # (이 로직은 복잡해질 수 있으므로, 원본처럼 start_cell 기준으로만 배치 시도)
                
                # 이 모양의 셀 좌표 계산
                group_cells = {(start_cell[0] + r_off, start_cell[1] + (c_off if is_left else -c_off))
                               for r_off, c_off in pattern}
                
                # 이 모양이 현재 남은 칸(current_unfilled)에 정확히 들어맞는지 확인
                if group_cells.issubset(current_unfilled):
                    
                    # (1D 모양에 대한 고립 셀 체크 - 원본 로직 유지)
                    shape_height = max(r_off for r_off, c_off in pattern) + 1
                    shape_width = max(c_off for r_off, c_off in pattern) + 1
                    if shape_height == 1 or shape_width == 1: # 1D 모양일 때만
                        remaining = current_unfilled - group_cells
                        is_isolated = True
                        for r_cell, c_cell in group_cells:
                            if (r_cell - 1, c_cell) in remaining or (r_cell + 1, c_cell) in remaining:
                                is_isolated = False
                                break
                        if not is_isolated:
                            continue # 고립된 셀을 만드므로 이 모양은 패스

                    # (통과) 재귀 호출
                    current_placements.append({'size': size, 'cells': group_cells, 'side': side})
                    _find_all_solutions_helper(current_unfilled - group_cells, current_placements)
                    current_placements.pop() # 백트래킹
            
            # (start_cell을 덮는 모양이 하나도 없으면 이 경로는 실패)

        # --- 래퍼 함수 메인 로직 ---
        _find_all_solutions_helper(unfilled, [])
        
        if not all_solutions:
            print(f"  > [실패] {side} 영역에서 해를 찾지 못했습니다.")
            return False # 해가 없음
        
        # (성공) 찾은 모든 해 중에서 '최소 조각 개수'를 가진 해를 찾음
        best_solution = min(all_solutions, key=len)
        print(f"  > [성공] {side} 영역: {len(all_solutions)}개 해 발견, 최소 조각 {len(best_solution)}개 선택")
        
        # 메인 placements 리스트에 최적의 해를 추가
        placements.extend(best_solution)
        return True

    # =============================================================
    # (이하 로직은 원본과 동일)
    # =============================================================
    def _rd_take_v10(self, need, target_counts, rd_remaining, rds, cursor):
        """RD 큐에서 RD 번호를 할당받는 로직"""
        n = len(rds)
        if n == 0: return None
        
        current_idx = cursor[0] 
        while current_idx < n:
            current_rd = rds[current_idx]
            
            can_fit = rd_remaining.get(current_rd, 0) >= need
            below_target = target_counts.get(current_rd, 0) > 0 
        
            if can_fit and below_target:
                rd_remaining[current_rd] -= need
                target_counts[current_rd] = target_counts.get(current_rd, 0) - need
                cursor[0] = current_idx 
                return current_rd
            current_idx += 1 
            
        for i in range(n): 
            check_idx = i
            check_rd = rds[check_idx]
            if rd_remaining.get(check_rd, 0) >= need:
                rd_remaining[check_rd] -= need
                target_counts[check_rd] = target_counts.get(check_rd, 0) - need 
                cursor[0] = check_idx
                return check_rd
                
        return None 

    def _pick_color(self, cells, b):
        """인접 셀과 겹치지 않는 색상을 선택"""
        palette = self.FIXED_COLORS 
        adj_colors = set()
        for r, c in cells:
            for dr, dc in [(0, 1), (0, -1), (1, 0), (-1, 0)]: 
                adj_cell = (r + dr, c + dc)
                if adj_cell in b.cell_colors: 
                    adj_colors.add(b.cell_colors[adj_cell]) 
        
        for color in palette:
            if color not in adj_colors:
                return color
        return palette[0]

# --------------------------------
# 앱 실행
# --------------------------------
class App(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Auto-Set Tester (Optimal / ⚠️Slow HATCH)") # 제목 변경
        try:
            from ctypes import windll; windll.shcore.SetProcessDpiAwareness(1)
        except: pass
        self.geometry("1100x700") 
        self.minsize(800, 500) 
        MinimalAutoSetTester(self)

if __name__ == '__main__':
    app = App() 
    app.mainloop()
