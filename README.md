# CHESS_ENGimport 
tkinter as tk
from tkinter import messagebox
import chess
import chess.engine
import threading
import atexit

# ✅ Correct Stockfish Path (Update this as needed)
STOCKFISH_PATH = "C:\\Users\\shiva\\Downloads\\stockfish-windows-x86-64-avx2\\stockfish\\stockfish-windows-x86-64-avx2.exe"

class ChessCoachApp:
    def __init__(self, master):
        self.master = master
        self.master.title("AI Chess Coach")
        self.board = chess.Board()
        self.buttons = {}
        self.selected_square = None
        self.last_move = None
        self.move_stack = []

        # Chess piece emojis
        self.piece_emojis = {
            'K': '♔', 'Q': '♕', 'R': '♖', 'N': '♘', 'B': '♗', 'P': '♙',
            'k': '♚', 'q': '♛', 'r': '♜', 'n': '♞', 'b': '♝', 'p': '♟'
        }

        self.highlight_color_from = "#90EE90"  # Light green
        self.highlight_color_to = "#ADD8E6"    # Light blue

        try:
            self.engine = chess.engine.SimpleEngine.popen_uci(STOCKFISH_PATH)
            atexit.register(self.engine.quit)
        except FileNotFoundError:
            messagebox.showerror("Error", "Stockfish file nahi mila. Path check karo.")
            self.master.destroy()
            return

        self.create_board()
        self.update_board()

    def create_board(self):
        files = "ABCDEFGH"

        # Column labels (A–H)
        for col in range(8):
            label = tk.Label(self.master, text=files[col], font=("Helvetica", 10, "bold"))
            label.grid(row=0, column=col + 1, padx=2, pady=2)

        # Row labels and board buttons
        for row in range(8):
            label = tk.Label(self.master, text=str(8 - row), font=("Helvetica", 10, "bold"))
            label.grid(row=row + 1, column=0, padx=2, pady=2)

            for col in range(8):
                square_name = chess.square_name(chess.square(col, 7 - row))
                button = tk.Button(self.master, width=4, height=2, font=("Helvetica", 16),
                                   command=lambda sq=square_name: self.on_square_click(sq))
                button.grid(row=row + 1, column=col + 1, padx=1, pady=1)
                self.buttons[square_name] = button

        # Info label and buttons
        self.info_label = tk.Label(self.master, text="Select a piece to move", font=("Helvetica", 12))
        self.info_label.grid(row=9, column=0, columnspan=9, pady=5)

        self.suggest_button = tk.Button(self.master, text="Suggest Move", command=self.suggest_move)
        self.suggest_button.grid(row=10, column=0, columnspan=3, pady=5)

        self.prev_button = tk.Button(self.master, text="Previous Move", command=self.prev_move)
        self.prev_button.grid(row=10, column=3, columnspan=3, pady=5)

        self.curr_button = tk.Button(self.master, text="Current Move", command=self.current_move)
        self.curr_button.grid(row=10, column=6, columnspan=3, pady=5)

        self.exit_button = tk.Button(self.master, text="Exit", command=self.exit_game,
                                     font=("Helvetica", 12, "bold"), bg="red", fg="white")
        self.exit_button.grid(row=11, column=0, columnspan=9, pady=5)

    def update_board(self):
        for square in chess.SQUARES:
            piece = self.board.piece_at(square)
            name = chess.square_name(square)
            color = "#F0D9B5" if (square + (square // 8)) % 2 == 0 else "#B58863"

            # Highlight last move
            if self.last_move:
                if square == self.last_move.from_square:
                    color = self.highlight_color_from
                elif square == self.last_move.to_square:
                    color = self.highlight_color_to

            symbol = piece.symbol() if piece else None
            if symbol:
                self.buttons[name].config(text=self.piece_emojis[symbol], bg=color)
            else:
                self.buttons[name].config(text='', bg=color)

    def on_square_click(self, square_name):
        square = chess.parse_square(square_name)
        if self.selected_square is None:
            if self.board.piece_at(square) and self.board.piece_at(square).color == self.board.turn:
                self.selected_square = square
                self.info_label.config(text=f"Selected {square_name}")
            else:
                self.info_label.config(text="Select a valid piece")
        else:
            move = chess.Move(self.selected_square, square)
            if move in self.board.legal_moves:
                self.last_move = move
                self.move_stack.append(self.board.copy())
                self.board.push(move)
                self.selected_square = None
                self.update_board()
                self.info_label.config(text="Move played")
                self.play_engine_move()
            else:
                self.info_label.config(text="Invalid move")
                self.selected_square = None

    def suggest_move(self):
        def get_suggestion():
            with self.engine.analysis(self.board, limit=chess.engine.Limit(time=1)) as analysis:
                for info in analysis:
                    if "pv" in info:
                        move = info["pv"][0]
                        break
            move_str = self.board.san(move)
            self.info_label.config(text=f"Suggested move: {move_str}")

        threading.Thread(target=get_suggestion).start()

    def play_engine_move(self):
        def make_move():
            if self.board.is_game_over():
                return
            self.move_stack.append(self.board.copy())
            result = self.engine.play(self.board, limit=chess.engine.Limit(time=1))
            self.last_move = result.move
            self.board.push(result.move)
            self.update_board()
            self.info_label.config(text=f"Engine played: {self.board.san(result.move)}")

        threading.Thread(target=make_move).start()

    def prev_move(self):
        if len(self.board.move_stack) >= 1:
            self.last_move = None
            self.board.pop()
            self.update_board()
            self.info_label.config(text="Went to previous move")

    def current_move(self):
        if self.move_stack:
            self.board = self.move_stack[-1].copy()
            self.update_board()
            self.info_label.config(text="Restored current move")

    def exit_game(self):
        if messagebox.askokcancel("Exit", "Do you want to quit the game?"):
            self.master.quit()

    def __del__(self):
        if hasattr(self, 'engine'):
            self.engine.quit()

if __name__ == "__main__":
    root = tk.Tk()
    app = ChessCoachApp(root)
    root.mainloop()
