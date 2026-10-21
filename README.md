	# -*- coding: utf-8 -*-
"""Minha Saúde Financeira — app pessoal em Python/Kivy, com SQLite local."""
import os, csv, sqlite3, calendar
from datetime import date, datetime
from pathlib import Path
from kivy.app import App
from kivy.metrics import dp
from kivy.core.window import Window
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.gridlayout import GridLayout
from kivy.uix.scrollview import ScrollView
from kivy.uix.label import Label
from kivy.uix.button import Button
from kivy.uix.textinput import TextInput
from kivy.uix.spinner import Spinner
from kivy.uix.popup import Popup
from kivy.graphics import Color, Rectangle, Line

CATEGORIES = ["Alimentação","Transporte","Moradia/Contas","Saúde/Farmácia","Faculdade/Estudos",
"Academia/Fitness","Cuidados pessoais","Lazer","Compras","Dívidas/Parcelas","Assinaturas",
"Trabalho","Renda/Salário","Renda extra","Investimentos/Poupança","Outros"]
PAYMENTS = ["Pix","Débito","Crédito","Dinheiro","Transferência","Boleto","Outro"]
GOAL_TYPES = ["Notebook","Carro","Reserva de emergência","Outro"]
def money(v):
    return ("R$ {:,.2f}".format(float(v or 0))).replace(",", "X").replace(".", ",").replace("X", ".")

class FinanceApp(App):
    def build(self):
        self.title = "Minha Saúde Financeira"
        self.db_path = os.path.join(self.user_data_dir, "financeiro.db")
        Window.softinput_mode = "below_target"
        self.init_db()
        root = BoxLayout(orientation="vertical", padding=dp(9), spacing=dp(7))
        root.add_widget(Label(text="[b]MINHA SAÚDE FINANCEIRA[/b]", markup=True, font_size="21sp",
                              size_hint_y=None, height=dp(38)))
        self.summary = Label(text="", markup=True, size_hint_y=None, height=dp(88), halign="left", valign="middle")
        self.summary.bind(size=lambda w,*_: setattr(w,"text_size",w.size)); root.add_widget(self.summary)
        buttons = GridLayout(cols=2, spacing=dp(5), size_hint_y=None, height=dp(145))
        for label, fn in [
            ("＋ Receita/despesa", self.add_transaction), ("🎯 Metas de economia", self.goals_screen),
            ("💳 Parcelas do cartão", self.card_screen), ("🔔 Vencimentos", self.bills_screen),
            ("📊 Gráficos de gastos", self.chart_screen), ("⬇ Exportar CSV", self.export_csv),
        ]:
            b=Button(text=label,font_size="13sp"); b.bind(on_release=lambda _,f=fn:f()); buttons.add_widget(b)
        root.add_widget(buttons)
        root.add_widget(Label(text="[b]Movimentações recentes[/b]",markup=True,size_hint_y=None,height=dp(26)))
        self.entries=GridLayout(cols=1,spacing=dp(4),size_hint_y=None); self.entries.bind(minimum_height=self.entries.setter("height"))
        sv=ScrollView(); sv.add_widget(self.entries); root.add_widget(sv)
        self.refresh()
        return root

    def db(self): return sqlite3.connect(self.db_path)
    def init_db(self):
        with self.db() as c:
            c.execute("""CREATE TABLE IF NOT EXISTS transactions(
            id INTEGER PRIMARY KEY,kind TEXT,amount REAL,category TEXT,tx_date TEXT,payment TEXT,description TEXT,attachment TEXT,created_at TEXT)""")
            c.execute("CREATE TABLE IF NOT EXISTS settings(key TEXT PRIMARY KEY,value TEXT)")
            c.execute("INSERT OR IGNORE INTO settings VALUES('monthly_budget','0')")
            c.execute("""CREATE TABLE IF NOT EXISTS goals(id INTEGER PRIMARY KEY,title TEXT,target REAL,saved REAL,target_date TEXT,notes TEXT)""")
            c.execute("""CREATE TABLE IF NOT EXISTS card_installments(id INTEGER PRIMARY KEY,title TEXT,total REAL,installment_value REAL,installments INTEGER,paid INTEGER,due_day INTEGER,card TEXT)""")
            c.execute("""CREATE TABLE IF NOT EXISTS bills(id INTEGER PRIMARY KEY,title TEXT,amount REAL,due_date TEXT,category TEXT,status TEXT,notes TEXT)""")

    def budget(self):
        with self.db() as c: r=c.execute("SELECT value FROM settings WHERE key='monthly_budget'").fetchone()
        return float(r[0]) if r else 0
    def totals(self, month=None):
        month=month or date.today().strftime("%Y-%m")
        with self.db() as c: rows=c.execute("SELECT kind,SUM(amount) FROM transactions WHERE substr(tx_date,1,7)=? GROUP BY kind",(month,)).fetchall()
        d={"Receita":0,"Despesa":0}
        for k,v in rows: d[k]=v or 0
        return d["Receita"],d["Despesa"]
    def refresh(self):
        inc,exp=self.totals(); bud=self.budget()
        text=f"[b]Este mês[/b]  Entradas: {money(inc)}   Saídas: {money(exp)}\\n[b]Saldo:[/b] {money(inc-exp)}"
        if bud: text+=f"\\nOrçamento: {money(bud)} | "+("Restante: "+money(bud-exp) if bud>=exp else "Excedido: "+money(exp-bud))
        self.summary.text=text
        self.entries.clear_widgets()
        with self.db() as c: rows=c.execute("SELECT id,kind,amount,category,tx_date,payment,description FROM transactions ORDER BY tx_date DESC,id DESC LIMIT 60").fetchall()
        if not rows: self.entries.add_widget(Label(text="Nenhum lançamento ainda.",size_hint_y=None,height=dp(48)))
        for r in rows:
            tid,kind,amt,cat,dt,pay,desc=r
            row=BoxLayout(size_hint_y=None,height=dp(64),spacing=dp(4))
            sign="+" if kind=="Receita" else "−"; col=(.1,.5,.25,1) if kind=="Receita" else (.7,.18,.18,1)
            info=Label(text=f"[b]{sign}{money(amt)} — {cat}[/b]\\n{dt} • {pay or ''} • {desc or ''}",markup=True,
                       color=col,halign="left",valign="middle")
            info.bind(size=lambda w,*_:setattr(w,"text_size",w.size)); row.add_widget(info)
            b=Button(text="Excluir",size_hint_x=None,width=dp(64)); b.bind(on_release=lambda _,x=tid:self.delete_tx(x)); row.add_widget(b)
            self.entries.add_widget(row)

    def popup(self,title,content,size=(.94,.88)):
        p=Popup(title=title,content=content,size_hint=size); p.open(); return p
    def message(self,msg):
        box=BoxLayout(orientation="vertical",padding=dp(10),spacing=dp(8)); lab=Label(text=msg,halign="left",valign="middle")
        lab.bind(size=lambda w,*_:setattr(w,"text_size",w.size)); box.add_widget(lab)
        b=Button(text="OK",size_hint_y=None,height=dp(42)); box.add_widget(b)
        p=self.popup("Minha Saúde Financeira",box,(.9,.45)); b.bind(on_release=lambda *_:p.dismiss())
    def field(self, hint, value="", height=42):
        return TextInput(hint_text=hint,text=str(value),multiline=False,size_hint_y=None,height=dp(height))
    def make_form(self,title,fields,save_fn):
        # fields = list of (label, widget)
        content=BoxLayout(orientation="vertical",padding=dp(9),spacing=dp(5))
        form=GridLayout(cols=1,spacing=dp(3),size_hint_y=None); form.bind(minimum_height=form.setter("height"))
        for lab,w in fields:
            form.add_widget(Label(text=lab,size_hint_y=None,height=dp(20),halign="left")); form.add_widget(w)
        sv=ScrollView(); sv.add_widget(form); content.add_widget(sv)
        save=Button(text="Salvar",size_hint_y=None,height=dp(45)); content.add_widget(save)
        p=self.popup(title,content); save.bind(on_release=lambda *_:save_fn(p)); return p
    def add_transaction(self,*_):
        kind=Spinner(text="Despesa",values=["Despesa","Receita"],size_hint_y=None,height=dp(42))
        amount=self.field("Valor (ex.: 35,90)"); cat=Spinner(text="Alimentação",values=CATEGORIES,size_hint_y=None,height=dp(42))
        dt=self.field("Data AAAA-MM-DD",date.today().isoformat()); pay=Spinner(text="Pix",values=PAYMENTS,size_hint_y=None,height=dp(42))
        desc=self.field("Descrição")
        def save(p):
            try:
                v=float(amount.text.replace("R$","").replace(" ","").replace(".","").replace(",","."))
                datetime.strptime(dt.text,"%Y-%m-%d")
                if v<=0: raise ValueError()
            except: self.message("Confira o valor e a data. Ex.: 35,90 e 2026-10-19."); return
            with self.db() as c: c.execute("INSERT INTO transactions(kind,amount,category,tx_date,payment,description,attachment,created_at) VALUES(?,?,?,?,?,?,?,?)",
                (kind.text,v,cat.text,dt.text,pay.text,desc.text,"",datetime.now().isoformat()))
            p.dismiss(); self.refresh()
        self.make_form("Nova movimentação",[("Tipo",kind),("Valor (R$)",amount),("Categoria",cat),("Data",dt),("Pagamento",pay),("Descrição",desc)],save)
    def delete_tx(self,tid):
        with self.db() as c:c.execute("DELETE FROM transactions WHERE id=?",(tid,))
        self.refresh()

    def goals_screen(self,*_):
        content=BoxLayout(orientation="vertical",padding=dp(8),spacing=dp(6))
        add=Button(text="＋ Criar meta",size_hint_y=None,height=dp(43)); content.add_widget(add)
        grid=GridLayout(cols=1,spacing=dp(5),size_hint_y=None); grid.bind(minimum_height=grid.setter("height"))
        with self.db() as c: goals=c.execute("SELECT id,title,target,saved,target_date,notes FROM goals ORDER BY id DESC").fetchall()
        if not goals: grid.add_widget(Label(text="Crie metas para notebook, carro e reserva de emergência.",size_hint_y=None,height=dp(55)))
        for g in goals:
            gid,title,target,saved,td,notes=g; remain=max(0,target-saved); pct=min(100,(saved/target*100 if target else 0))
            box=BoxLayout(orientation="vertical",size_hint_y=None,height=dp(94),padding=dp(4))
            box.add_widget(Label(text=f"[b]{title}[/b] — {pct:.1f}% concluído",markup=True,size_hint_y=.32))
            box.add_widget(Label(text=f"Guardado: {money(saved)} de {money(target)} | Falta: {money(remain)}\\nPrazo: {td or 'Não definido'}",size_hint_y=.4))
            row=BoxLayout(size_hint_y=.3,spacing=dp(4)); plus=Button(text="＋ Registrar economia"); delete=Button(text="Excluir")
            plus.bind(on_release=lambda _,x=g:self.add_saving(x,content)); delete.bind(on_release=lambda _,x=gid:self.delete_goal(x,content))
            row.add_widget(plus);row.add_widget(delete);box.add_widget(row);grid.add_widget(box)
        sc=ScrollView();sc.add_widget(grid);content.add_widget(sc)
        close=Button(text="Fechar",size_hint_y=None,height=dp(40));content.add_widget(close)
        p=self.popup("Metas de economia",content);close.bind(on_release=lambda *_:p.dismiss())
        add.bind(on_release=lambda *_:self.goal_form(p))
    def goal_form(self,parent):
        title=Spinner(text="Notebook",values=GOAL_TYPES,size_hint_y=None,height=dp(42))
        custom=self.field("Nome da meta (opcional)")
        target=self.field("Valor total desejado (R$)"); saved=self.field("Quanto já guardou? (R$)","0")
        dt=self.field("Prazo AAAA-MM-DD (opcional)"); notes=self.field("Observações")
        def save(p):
            try:
                t=float(target.text.replace(".","").replace(",","."))
                s=float(saved.text.replace(".","").replace(",","."))
                if t<=0 or s<0: raise ValueError()
                if dt.text.strip(): datetime.strptime(dt.text.strip(),"%Y-%m-%d")
            except: self.message("Confira os valores e o prazo.");return
            name=custom.text.strip() or title.text
            with self.db() as c:c.execute("INSERT INTO goals(title,target,saved,target_date,notes) VALUES(?,?,?,?,?)",(name,t,s,dt.text.strip(),notes.text.strip()))
            p.dismiss();self.goals_screen()
        self.make_form("Criar meta",[("Tipo",title),("Nome opcional",custom),("Valor-alvo",target),("Já guardado",saved),("Prazo",dt),("Observações",notes)],save)
    def add_saving(self,g,old_popup):
        amt=self.field("Quanto você guardou? (R$)")
        def save(p):
            try:v=float(amt.text.replace(".","").replace(",","."))
            except:self.message("Digite um valor válido.");return
            if v<=0:self.message("Informe um valor maior que zero.");return
            with self.db() as c:c.execute("UPDATE goals SET saved=saved+? WHERE id=?",(v,g[0]))
            p.dismiss();old_popup.dismiss();self.goals_screen()
        self.make_form("Registrar economia",[("Valor guardado",amt)],save)
    def delete_goal(self,gid,parent):
        with self.db() as c:c.execute("DELETE FROM goals WHERE id=?",(gid,))
        parent.dismiss();self.goals_screen()

    def card_screen(self,*_):
        content=BoxLayout(orientation="vertical",padding=dp(8),spacing=dp(6))
        add=Button(text="＋ Adicionar compra parcelada",size_hint_y=None,height=dp(44));content.add_widget(add)
        grid=GridLayout(cols=1,spacing=dp(4),size_hint_y=None);grid.bind(minimum_height=grid.setter("height"))
        with self.db() as c: rows=c.execute("SELECT id,title,total,installment_value,installments,paid,due_day,card FROM card_installments ORDER BY due_day").fetchall()
        if not rows:grid.add_widget(Label(text="Nenhuma compra parcelada cadastrada.",size_hint_y=None,height=dp(48)))
        for r in rows:
            rid,title,total,val,n,paid,day,card=r
            box=BoxLayout(orientation="vertical",size_hint_y=None,height=dp(90))
            box.add_widget(Label(text=f"[b]{title}[/b] ({card or 'Cartão'})",markup=True,size_hint_y=.35))
            box.add_widget(Label(text=f"{paid}/{n} parcelas pagas • {money(val)}/mês • Total {money(total)}\\nVencimento: dia {day} | Restante: {money(val*(n-paid))}",size_hint_y=.4))
            row=BoxLayout(size_hint_y=.3,spacing=dp(4));btn=Button(text="Marcar parcela paga");dele=Button(text="Excluir")
            btn.disabled=paid>=n
            btn.bind(on_release=lambda _,x=rid:self.pay_installment(x));dele.bind(on_release=lambda _,x=rid:self.delete_installment(x))
            row.add_widget(btn);row.add_widget(dele);box.add_widget(row);grid.add_widget(box)
        sc=ScrollView();sc.add_widget(grid);content.add_widget(sc)
        close=Button(text="Fechar",size_hint_y=None,height=dp(40));content.add_widget(close)
        p=self.popup("Parcelas do cartão",content);close.bind(on_release=lambda *_:p.dismiss())
        add.bind(on_release=lambda *_:self.card_form(p))
    def card_form(self,parent):
        title=self.field("Nome da compra (ex.: notebook)")
        total=self.field("Valor total (R$)"); val=self.field("Valor de cada parcela (R$)")
        n=self.field("Quantidade total de parcelas"); paid=self.field("Parcelas já pagas","0")
        day=self.field("Dia do vencimento (1 a 31)","10"); card=self.field("Nome do cartão")
        def save(p):
            try:
                t=float(total.text.replace(".","").replace(",","."))
                v=float(val.text.replace(".","").replace(",","."))
                nn=int(n.text); pp=int(paid.text); dd=int(day.text)
                if t<=0 or v<=0 or nn<1 or pp<0 or pp>nn or dd<1 or dd>31:raise ValueError()
            except:self.message("Confira valores, parcelas e dia de vencimento.");return
            with self.db() as c:c.execute("INSERT INTO card_installments(title,total,installment_value,installments,paid,due_day,card) VALUES(?,?,?,?,?,?,?)",(title.text or "Compra parcelada",t,v,nn,pp,dd,card.text))
            p.dismiss();self.card_screen()
        self.make_form("Nova compra parcelada",[("Compra",title),("Valor total",total),("Valor da parcela",val),("Número de parcelas",n),("Já pagas",paid),("Vencimento (dia)",day),("Cartão",card)],save)
    def pay_installment(self,rid):
        with self.db() as c:c.execute("UPDATE card_installments SET paid=MIN(paid+1,installments) WHERE id=?",(rid,))
        self.card_screen()
    def delete_installment(self,rid):
        with self.db() as c:c.execute("DELETE FROM card_installments WHERE id=?",(rid,))
        self.card_screen()

    def bills_screen(self,*_):
        content=BoxLayout(orientation="vertical",padding=dp(8),spacing=dp(6))
        add=Button(text="＋ Cadastrar conta",size_hint_y=None,height=dp(43));content.add_widget(add)
        grid=GridLayout(cols=1,spacing=dp(4),size_hint_y=None);grid.bind(minimum_height=grid.setter("height"))
        with self.db() as c: bills=c.execute("SELECT id,title,amount,due_date,category,status,notes FROM bills ORDER BY status,due_date").fetchall()
        today=date.today().isoformat()
        if not bills:grid.add_widget(Label(text="Cadastre contas fixas, faturas e vencimentos.",size_hint_y=None,height=dp(50)))
        for b in bills:
            bid,title,amt,dt,cat,status,notes=b
            state="PAGA" if status=="Paga" else ("ATRASADA" if dt<today else "PENDENTE")
            box=BoxLayout(orientation="vertical",size_hint_y=None,height=dp(82))
            box.add_widget(Label(text=f"[b]{title} — {money(amt)}[/b] [{state}]",markup=True,size_hint_y=.38))
            box.add_widget(Label(text=f"Vence: {dt} • {cat} • {notes or ''}",size_hint_y=.3))
            row=BoxLayout(size_hint_y=.32,spacing=dp(4));toggle=Button(text="Marcar como "+("pendente" if status=="Paga" else "paga"));dele=Button(text="Excluir")
            toggle.bind(on_release=lambda _,x=bid,s=status:self.toggle_bill(x,s));dele.bind(on_release=lambda _,x=bid:self.delete_bill(x))
            row.add_widget(toggle);row.add_widget(dele);box.add_widget(row);grid.add_widget(box)
        sc=ScrollView();sc.add_widget(grid);content.add_widget(sc)
        close=Button(text="Fechar",size_hint_y=None,height=dp(40));content.add_widget(close)
        p=self.popup("Vencimentos de contas",content);close.bind(on_release=lambda *_:p.dismiss())
        add.bind(on_release=lambda *_:self.bill_form(p))
    def bill_form(self,parent):
        title=self.field("Nome da conta (ex.: internet, energia)")
        amount=self.field("Valor (R$)");dt=self.field("Vencimento AAAA-MM-DD",date.today().isoformat())
        cat=Spinner(text="Moradia/Contas",values=CATEGORIES,size_hint_y=None,height=dp(42));notes=self.field("Observações")
        def save(p):
            try:
                v=float(amount.text.replace(".","").replace(",","."))
                datetime.strptime(dt.text,"%Y-%m-%d")
                if v<=0:raise ValueError()
            except:self.message("Confira o valor e a data.");return
            with self.db() as c:c.execute("INSERT INTO bills(title,amount,due_date,category,status,notes) VALUES(?,?,?,?,?,?)",(title.text or "Conta",v,dt.text,cat.text,"Pendente",notes.text))
            p.dismiss();self.bills_screen()
        self.make_form("Cadastrar vencimento",[("Conta",title),("Valor",amount),("Data de vencimento",dt),("Categoria",cat),("Observações",notes)],save)
    def toggle_bill(self,bid,status):
        new="Pendente" if status=="Paga" else "Paga"
        with self.db() as c:c.execute("UPDATE bills SET status=? WHERE id=?",(new,bid))
        self.bills_screen()
    def delete_bill(self,bid):
        with self.db() as c:c.execute("DELETE FROM bills WHERE id=?",(bid,))
        self.bills_screen()

    def chart_screen(self,*_):
        month=date.today().strftime("%Y-%m")
        with self.db() as c:
            rows=c.execute("SELECT category,SUM(amount) FROM transactions WHERE kind='Despesa' AND substr(tx_date,1,7)=? GROUP BY category ORDER BY SUM(amount) DESC",(month,)).fetchall()
        box=BoxLayout(orientation="vertical",padding=dp(10),spacing=dp(6))
        box.add_widget(Label(text=f"[b]Gastos por categoria — {month}[/b]",markup=True,size_hint_y=None,height=dp(34)))
        if not rows:
            box.add_widget(Label(text="Ainda não há despesas neste mês. Cadastre movimentações para gerar o gráfico."))
        else:
            maxv=max(v for _,v in rows) or 1
            for cat,val in rows:
                row=BoxLayout(size_hint_y=None,height=dp(38),spacing=dp(5))
                label=Label(text=f"{cat}\\n{money(val)}",halign="left",valign="middle",size_hint_x=.46,font_size="12sp")
                label.bind(size=lambda w,*_:setattr(w,"text_size",w.size));row.add_widget(label)
                bar=BoxLayout(size_hint_x=.54)
                with bar.canvas:
                    Color(.22,.55,.78,1); rect=Rectangle(pos=bar.pos,size=(1,dp(22)))
                def upd(widget,*_,v=val,m=maxv,rect=rect):
                    rect.pos=(widget.x,widget.center_y-dp(11));rect.size=(max(dp(2),widget.width*v/m),dp(22))
                bar.bind(pos=upd,size=upd);upd(bar)
                row.add_widget(bar);box.add_widget(row)
        close=Button(text="Fechar",size_hint_y=None,height=dp(42));box.add_widget(close)
        p=self.popup("Gráfico de gastos",box);close.bind(on_release=lambda *_:p.dismiss())

    def export_csv(self,*_):
        try:
            path=os.path.join(self.user_data_dir,"saude_financeira_export.csv")
            with self.db() as c,open(path,"w",newline="",encoding="utf-8-sig") as f:
                w=csv.writer(f,delimiter=";")
                w.writerow(["Tabela","ID","Nome/Tipo","Valor","Categoria","Data","Detalhes"])
                for r in c.execute("SELECT id,kind,amount,category,tx_date,payment,description FROM transactions"):
                    w.writerow(["Movimentação",*r])
                for r in c.execute("SELECT id,title,target,saved,target_date,notes FROM goals"):
                    w.writerow(["Meta",*r])
                for r in c.execute("SELECT id,title,total,installment_value,installments,paid,due_day,card FROM card_installments"):
                    w.writerow(["Cartão",*r])
                for r in c.execute("SELECT id,title,amount,due_date,category,status,notes FROM bills"):
                    w.writerow(["Conta",*r])
            self.message("Exportação criada em:\\n"+path+"\\nFaça cópias de segurança regularmente.")
        except Exception as e:self.message("Erro ao exportar: "+str(e))

if __name__=="__main__":
    FinanceApp().run()
