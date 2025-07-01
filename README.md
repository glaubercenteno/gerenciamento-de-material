# gerenciamento-de-material
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///monitoramento.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# Modelos
class Material(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    nome = db.Column(db.String(100), unique=True, nullable=False)
    descricao = db.Column(db.String(200))

class Setor(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    nome = db.Column(db.String(100), unique=True, nullable=False)

class Consumo(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    material_id = db.Column(db.Integer, db.ForeignKey('material.id'), nullable=False)
    setor_id = db.Column(db.Integer, db.ForeignKey('setor.id'), nullable=False)
    quantidade = db.Column(db.Float, nullable=False)
    data = db.Column(db.DateTime, default=datetime.utcnow)
    material = db.relationship('Material')
    setor = db.relationship('Setor')

# Criar tabelas antes do primeiro request
@app.before_first_request
def create_tables():
    db.create_all()

# Rota para cadastrar material
@app.route('/materiais', methods=['POST'])
def cadastrar_material():
    data = request.get_json()
    material = Material(nome=data['nome'], descricao=data.get('descricao', ''))
    db.session.add(material)
    db.session.commit()
    return jsonify({'message': 'Material cadastrado com sucesso.'}), 201

# Rota para cadastrar setor
@app.route('/setores', methods=['POST'])
def cadastrar_setor():
    data = request.get_json()
    setor = Setor(nome=data['nome'])
    db.session.add(setor)
    db.session.commit()
    return jsonify({'message': 'Setor cadastrado com sucesso.'}), 201

# Rota para registrar consumo
@app.route('/consumos', methods=['POST'])
def registrar_consumo():
    data = request.get_json()
    data_consumo = datetime.strptime(data['data'], "%Y-%m-%d %H:%M:%S") if data.get('data') else datetime.utcnow()
    consumo = Consumo(
        material_id=data['material_id'],
        setor_id=data['setor_id'],
        quantidade=data['quantidade'],
        data=data_consumo
    )
    db.session.add(consumo)
    db.session.commit()
    return jsonify({'message': 'Consumo registrado com sucesso.'}), 201

# Rota para relatório
@app.route('/relatorio', methods=['GET'])
def relatorio():
    material_id = request.args.get('material_id')
    setor_id = request.args.get('setor_id')
    data_inicio = request.args.get('data_inicio')
    data_fim = request.args.get('data_fim')

    query = Consumo.query
    if material_id:
        query = query.filter_by(material_id=material_id)
    if setor_id:
        query = query.filter_by(setor_id=setor_id)
    if data_inicio:
        query = query.filter(Consumo.data >= datetime.strptime(data_inicio, "%Y-%m-%d"))
    if data_fim:
        query = query.filter(Consumo.data <= datetime.strptime(data_fim, "%Y-%m-%d"))

    consumos = query.all()
    result = []
    for c in consumos:
        result.append({
            'material': c.material.nome,
            'setor': c.setor.nome,
            'quantidade': c.quantidade,
            'data': c.data.strftime("%Y-%m-%d %H:%M:%S")
        })

    return jsonify(result)

# Rodar o app
if __name__ == '__main__':
    app.run(debug=True)
