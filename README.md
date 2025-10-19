import tkinter as tk
from tkinter import scrolledtext, ttk, messagebox
import datetime
import threading
import requests
import json
from typing import List, Dict

class AIAssistant:
    def __init__(self, root):
        self.root = root
        self.root.title("🤖 Yapay Zeka Asistanı")
        self.root.geometry("900x700")
        self.root.configure(bg="#1e1e2e")
        self.root.minsize(600, 400)
        
        # API Ayarları
        self.api_key = "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"
        self.api_url = "https://api.groq.com/openai/v1/chat/completions"
        self.model = "llama-3.3-70b-versatile"
        
        # Konuşma geçmişi
        self.conversation_history: List[Dict[str, str]] = []
        self.max_history = 20  # Maksimum geçmiş mesaj sayısı
        
        # Durum
        self.is_processing = False
        
        # Arayüzü oluştur
        self.setup_styles()
        self.create_widgets()
        self.add_welcome_message()
        
    def setup_styles(self):
        """Gelişmiş stil ayarları"""
        style = ttk.Style()
        style.theme_use('clam')
        
        # Progress bar stili
        style.configure(
            "Custom.Horizontal.TProgressbar",
            troughcolor='#2b2b3c',
            background='#00b4d8',
            borderwidth=0,
            thickness=3
        )
        
    def create_widgets(self):
        """Gelişmiş arayüz bileşenleri"""
        
        # Başlık çerçevesi
        header_frame = tk.Frame(self.root, bg="#00b4d8", height=70)
        header_frame.pack(fill=tk.X, side=tk.TOP)
        header_frame.pack_propagate(False)
        
        # Başlık
        title_label = tk.Label(
            header_frame,
            text="🤖 YAPAY ZEKA ASİSTANI",
            font=("Segoe UI", 20, "bold"),
            bg="#00b4d8",
            fg="white"
        )
        title_label.pack(side=tk.LEFT, padx=20, pady=15)
        
        # Model bilgisi
        model_label = tk.Label(
            header_frame,
            text=f"Model: {self.model}",
            font=("Segoe UI", 9),
            bg="#00b4d8",
            fg="#e0f7ff"
        )
        model_label.pack(side=tk.RIGHT, padx=20)
        
        # Temizle butonu
        clear_button = tk.Button(
            header_frame,
            text="🗑️ Temizle",
            font=("Segoe UI", 10),
            bg="#0096c7",
            fg="white",
            relief=tk.FLAT,
            cursor="hand2",
            padx=15,
            pady=5,
            command=self.clear_chat
        )
        clear_button.pack(side=tk.RIGHT, padx=5)
        
        # Ana içerik çerçevesi
        main_frame = tk.Frame(self.root, bg="#1e1e2e")
        main_frame.pack(fill=tk.BOTH, expand=True, padx=15, pady=10)
        
        # Sohbet alanı çerçevesi
        chat_frame = tk.Frame(main_frame, bg="#1e1e2e")
        chat_frame.pack(fill=tk.BOTH, expand=True)
        
        # Sohbet görüntüleme alanı
        self.chat_display = scrolledtext.ScrolledText(
            chat_frame,
            wrap=tk.WORD,
            font=("Segoe UI", 11),
            bg="#2b2b3c",
            fg="#ffffff",
            insertbackground="white",
            relief=tk.FLAT,
            padx=15,
            pady=15,
            borderwidth=0,
            highlightthickness=1,
            highlightbackground="#00b4d8",
            highlightcolor="#00b4d8"
        )
        self.chat_display.pack(fill=tk.BOTH, expand=True)
        self.chat_display.config(state=tk.DISABLED)
        
        # Progress bar (yükleme göstergesi)
        self.progress_frame = tk.Frame(main_frame, bg="#1e1e2e", height=5)
        self.progress_frame.pack(fill=tk.X, pady=(5, 0))
        
        self.progress = ttk.Progressbar(
            self.progress_frame,
            style="Custom.Horizontal.TProgressbar",
            mode='indeterminate',
            length=300
        )
        
        # Durum çubuğu
        status_frame = tk.Frame(main_frame, bg="#1e1e2e", height=25)
        status_frame.pack(fill=tk.X, pady=(5, 0))
        status_frame.pack_propagate(False)
        
        self.status_label = tk.Label(
            status_frame,
            text="✓ Hazır",
            font=("Segoe UI", 9),
            bg="#1e1e2e",
            fg="#888888",
            anchor=tk.W
        )
        self.status_label.pack(side=tk.LEFT, fill=tk.X, expand=True)
        
        # Karakter sayacı
        self.char_count_label = tk.Label(
            status_frame,
            text="0/2000",
            font=("Segoe UI", 9),
            bg="#1e1e2e",
            fg="#888888",
            anchor=tk.E
        )
        self.char_count_label.pack(side=tk.RIGHT)
        
        # Giriş alanı çerçevesi
        input_frame = tk.Frame(main_frame, bg="#1e1e2e")
        input_frame.pack(fill=tk.X, pady=(10, 0))
        
        # Mesaj giriş çerçevesi (iç çerçeve)
        entry_container = tk.Frame(input_frame, bg="#2b2b3c", highlightthickness=2, 
                                   highlightbackground="#00b4d8", highlightcolor="#00b4d8")
        entry_container.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=(0, 10))
        
        # Mesaj giriş alanı (Text widget ile çok satırlı)
        self.message_entry = tk.Text(
            entry_container,
            font=("Segoe UI", 11),
            bg="#2b2b3c",
            fg="white",
            insertbackground="white",
            relief=tk.FLAT,
            borderwidth=0,
            height=3,
            wrap=tk.WORD,
            padx=10,
            pady=10
        )
        self.message_entry.pack(fill=tk.BOTH, expand=True)
        self.message_entry.bind("<KeyRelease>", self.update_char_count)
        self.message_entry.bind("<Control-Return>", lambda e: self.send_message())
        self.message_entry.focus()
        
        # Buton çerçevesi
        button_frame = tk.Frame(input_frame, bg="#1e1e2e")
        button_frame.pack(side=tk.RIGHT)
        
        # Gönder butonu
        self.send_button = tk.Button(
            button_frame,
            text="➤\nGÖNDER",
            font=("Segoe UI", 10, "bold"),
            bg="#00b4d8",
            fg="white",
            relief=tk.FLAT,
            cursor="hand2",
            padx=20,
            pady=15,
            command=self.send_message
        )
        self.send_button.pack()
        
        # Hover efektleri
        self.send_button.bind("<Enter>", lambda e: self.send_button.config(bg="#0096c7"))
        self.send_button.bind("<Leave>", lambda e: self.send_button.config(bg="#00b4d8"))
        
        # Yardım metni
        help_label = tk.Label(
            main_frame,
            text="💡 İpucu: Enter ile yeni satır, Ctrl+Enter ile gönder",
            font=("Segoe UI", 8),
            bg="#1e1e2e",
            fg="#555555"
        )
        help_label.pack(pady=(5, 0))
        
    def add_welcome_message(self):
        """Hoş geldin mesajı"""
        welcome = """Merhaba! Ben gelişmiş yapay zeka asistanınızım. 🎉

Size nasıl yardımcı olabilirim?
• Sorularınızı cevaplayabilirim
• Metin yazabilirim
• Kod önerileri sunabilirim
• Genel konularda bilgi verebilirim

Soru sormaktan çekinmeyin!"""
        
        self.add_message("Asistan", welcome, "#00b4d8")
        
    def update_char_count(self, event=None):
        """Karakter sayacını güncelle"""
        text = self.message_entry.get("1.0", tk.END).strip()
        char_count = len(text)
        self.char_count_label.config(text=f"{char_count}/2000")
        
        if char_count > 2000:
            self.char_count_label.config(fg="#ff4444")
        else:
            self.char_count_label.config(fg="#888888")
            
    def add_message(self, sender: str, message: str, color: str):
        """Gelişmiş mesaj ekleme"""
        self.chat_display.config(state=tk.NORMAL)
        
        # Zaman damgası
        timestamp = datetime.datetime.now().strftime("%H:%M:%S")
        
        # Ayırıcı çizgi (ilk mesaj değilse)
        current_text = self.chat_display.get("1.0", tk.END).strip()
        if current_text:
            self.chat_display.insert(tk.END, "\n" + "─" * 80 + "\n\n", ("separator",))
        
        # Başlık satırı
        self.chat_display.insert(tk.END, f"{sender} ", ("sender",))
        self.chat_display.insert(tk.END, f"• {timestamp}\n", ("time",))
        
        # Mesaj içeriği
        self.chat_display.insert(tk.END, f"{message}\n", ("message",))
        
        # Stil ayarları
        self.chat_display.tag_config("sender", foreground=color, font=("Segoe UI", 12, "bold"))
        self.chat_display.tag_config("time", foreground="#888888", font=("Segoe UI", 9))
        self.chat_display.tag_config("message", foreground="#ffffff", font=("Segoe UI", 11), spacing1=5)
        self.chat_display.tag_config("separator", foreground="#404050")
        
        self.chat_display.config(state=tk.DISABLED)
        self.chat_display.see(tk.END)
        
    def update_status(self, text: str, color: str = "#888888"):
        """Durum çubuğunu güncelle"""
        self.status_label.config(text=text, fg=color)
        self.root.update_idletasks()
        
    def show_progress(self):
        """Yükleme göstergesini göster"""
        self.progress.pack(fill=tk.X)
        self.progress.start(10)
        
    def hide_progress(self):
        """Yükleme göstergesini gizle"""
        self.progress.stop()
        self.progress.pack_forget()
        
    def clear_chat(self):
        """Sohbeti temizle"""
        if messagebox.askyesno("Temizle", "Tüm sohbet geçmişi silinecek. Emin misiniz?"):
            self.chat_display.config(state=tk.NORMAL)
            self.chat_display.delete("1.0", tk.END)
            self.chat_display.config(state=tk.DISABLED)
            self.conversation_history.clear()
            self.add_welcome_message()
            self.update_status("✓ Sohbet temizlendi", "#00ff88")
            
    def get_ai_response(self, user_input: str) -> str:
        """Groq API'den yanıt al - gelişmiş hata yönetimi"""
        try:
            # İlk mesajsa sistem mesajı ekle
            if len(self.conversation_history) == 0:
                self.conversation_history.append({
                    "role": "system",
                    "content": """Sen Türkçe konuşan, profesyonel ve yardımsever bir yapay zeka asistanısın. 
                    Özelliklerin:
                    - Her zaman Türkçe cevap verirsin
                    - Detaylı, anlaşılır ve yapılandırılmış yanıtlar verirsin
                    - Bilmediğin konularda dürüstçe söylersin
                    - Kullanıcıya saygılı ve arkadaş canlısı yaklaşırsın
                    - Karmaşık konuları basit örneklerle açıklarsın"""
                })
            
            # Kullanıcı mesajını ekle
            self.conversation_history.append({
                "role": "user",
                "content": user_input
            })
            
            # Geçmiş çok uzunsa eski mesajları sil (sistem mesajı hariç)
            if len(self.conversation_history) > self.max_history:
                self.conversation_history = [self.conversation_history[0]] + self.conversation_history[-(self.max_history-1):]
            
            # API isteği hazırla
            headers = {
                "Authorization": f"Bearer {self.api_key}",
                "Content-Type": "application/json"
            }
            
            data = {
                "model": self.model,
                "messages": self.conversation_history,
                "temperature": 0.7,
                "max_tokens": 2048,
                "top_p": 0.9,
                "stream": False
            }
            
            # İstek gönder
            response = requests.post(
                self.api_url, 
                headers=headers, 
                json=data, 
                timeout=45
            )
            
            # Yanıt kontrolü
            if response.status_code == 200:
                result = response.json()
                ai_message = result['choices'][0]['message']['content']
                
                # Yanıtı geçmişe ekle
                self.conversation_history.append({
                    "role": "assistant",
                    "content": ai_message
                })
                
                return ai_message
                
            elif response.status_code == 401:
                self._remove_last_user_message()
                return "❌ **Yetkilendirme Hatası**\n\nAPI anahtarınız geçersiz veya süresi dolmuş. Lütfen API anahtarınızı kontrol edin."
                
            elif response.status_code == 429:
                self._remove_last_user_message()
                return "⏳ **Oran Sınırı Aşıldı**\n\nÇok fazla istek gönderildi. Lütfen 1-2 dakika bekleyip tekrar deneyin."
                
            elif response.status_code == 500:
                self._remove_last_user_message()
                return "🔧 **Sunucu Hatası**\n\nGroq sunucularında geçici bir sorun var. Lütfen birkaç saniye sonra tekrar deneyin."
                
            else:
                self._remove_last_user_message()
                try:
                    error_detail = response.json().get('error', {})
                    error_msg = error_detail.get('message', 'Bilinmeyen hata')
                    error_type = error_detail.get('type', 'unknown')
                    return f"⚠️ **API Hatası ({response.status_code})**\n\nTip: {error_type}\nMesaj: {error_msg}"
                except:
                    return f"⚠️ **HTTP Hatası**\n\nDurum kodu: {response.status_code}\nLütfen daha sonra tekrar deneyin."
                    
        except requests.exceptions.Timeout:
            self._remove_last_user_message()
            return "⏱️ **Zaman Aşımı**\n\nSunucu yanıt vermedi (45 saniye). İnternet bağlantınız yavaş olabilir veya sunucu yoğun olabilir."
            
        except requests.exceptions.ConnectionError:
            self._remove_last_user_message()
            return "🌐 **Bağlantı Hatası**\n\nİnternet bağlantısı kurulamadı. Lütfen:\n• İnternet bağlantınızı kontrol edin\n• Güvenlik duvarı ayarlarınızı kontrol edin\n• VPN kullanıyorsanız kapatmayı deneyin"
            
        except json.JSONDecodeError:
            self._remove_last_user_message()
            return "⚠️ **Veri Formatı Hatası**\n\nSunucudan geçersiz yanıt alındı. Bu genellikle geçici bir sorundur, lütfen tekrar deneyin."
            
        except KeyError as e:
            self._remove_last_user_message()
            return f"⚠️ **Yanıt Ayrıştırma Hatası**\n\nBeklenen veri bulunamadı: {str(e)}\nLütfen tekrar deneyin."
            
        except Exception as e:
            self._remove_last_user_message()
            return f"❌ **Beklenmeyen Hata**\n\n{type(e).__name__}: {str(e)}\n\nBu hatayı görmemeliydiniz. Lütfen uygulamayı yeniden başlatın."
            
    def _remove_last_user_message(self):
        """Hata durumunda son kullanıcı mesajını geçmişten çıkar"""
        if self.conversation_history and self.conversation_history[-1]["role"] == "user":
            self.conversation_history.pop()
            
    def send_message(self):
        """Mesaj gönder - gelişmiş versiyon"""
        # Zaten işlem yapılıyorsa çık
        if self.is_processing:
            return
            
        user_message = self.message_entry.get("1.0", tk.END).strip()
        
        if not user_message:
            messagebox.showwarning("Uyarı", "Lütfen bir mesaj yazın!")
            return
            
        if len(user_message) > 2000:
            messagebox.showerror("Hata", "Mesajınız çok uzun! Maksimum 2000 karakter olabilir.")
            return
        
        # İşlem başlat
        self.is_processing = True
        self.send_button.config(state=tk.DISABLED, text="⏳\nGÖNDERİLİYOR")
        self.message_entry.config(state=tk.DISABLED)
        self.update_status("⏳ Gönderiliyor...", "#ffaa00")
        self.show_progress()
        
        # Kullanıcı mesajını göster
        self.add_message("Siz", user_message, "#00ff88")
        
        # Giriş alanını temizle
        self.message_entry.delete("1.0", tk.END)
        self.update_char_count()
        
        # AI yanıtını thread'de al
        thread = threading.Thread(target=self.process_ai_response, args=(user_message,), daemon=True)
        thread.start()
        
    def process_ai_response(self, user_message: str):
        """AI yanıtını işle - thread içinde çalışır"""
        try:
            self.root.after(0, lambda: self.update_status("🤔 Düşünüyor...", "#ffaa00"))
            
            # AI yanıtını al
            ai_response = self.get_ai_response(user_message)
            
            # Ana thread'de UI güncelle
            self.root.after(0, lambda: self.add_message("Asistan", ai_response, "#00b4d8"))
            self.root.after(0, lambda: self.update_status("✓ Yanıt alındı", "#00ff88"))
            
        except Exception as e:
            error_msg = f"❌ **Kritik Hata**\n\n{type(e).__name__}: {str(e)}"
            self.root.after(0, lambda: self.add_message("Sistem", error_msg, "#ff4444"))
            self.root.after(0, lambda: self.update_status("✗ Hata oluştu", "#ff4444"))
            
        finally:
            # UI'yi tekrar aktif et
            self.root.after(0, self.reset_ui)
            
    def reset_ui(self):
        """UI'yi varsayılan duruma getir"""
        self.is_processing = False
        self.hide_progress()
        self.send_button.config(state=tk.NORMAL, text="➤\nGÖNDER")
        self.message_entry.config(state=tk.NORMAL)
        self.message_entry.focus()
        self.root.after(3000, lambda: self.update_status("✓ Hazır", "#888888"))

def main():
    """Uygulamayı başlat"""
    root = tk.Tk()
    
    # Uygulama simgesi (opsiyonel)
    try:
        root.iconbitmap('icon.ico')
    except:
        pass
    
    # Uygulama oluştur
    app = AIAssistant(root)
    
    # Pencere kapatma onayı
    def on_closing():
        if messagebox.askokcancel("Çıkış", "Uygulamadan çıkmak istiyor musunuz?"):
            root.quit()
            
    root.protocol("WM_DELETE_WINDOW", on_closing)
    
    # Uygulamayı çalıştır
    root.mainloop()

if __name__ == "__main__":
    main()
