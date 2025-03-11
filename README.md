from flask import Flask, request, jsonify, send_file
from flask_sqlalchemy import SQLAlchemy
import pdfkit
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///snags.db'
db = SQLAlchemy(app)

class Snag(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    client_name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.String(255), nullable=False)
    image_path = db.Column(db.String(255), nullable=True)

@app.route('/add_snag', methods=['POST'])
def add_snag():
    data = request.form
    file = request.files.get('image')
    image_path = None
    
    if file:
        image_path = os.path.join('uploads', file.filename)
        file.save(image_path)
    
    new_snag = Snag(client_name=data['client_name'], description=data['description'], image_path=image_path)
    db.session.add(new_snag)
    db.session.commit()
    return jsonify({'message': 'Snag added successfully'}), 201

@app.route('/get_snags', methods=['GET'])
def get_snags():
    snags = Snag.query.all()
    result = [{'id': s.id, 'client_name': s.client_name, 'description': s.description, 'image_path': s.image_path} for s in snags]
    return jsonify(result)

@app.route('/generate_pdf', methods=['GET'])
def generate_pdf():
    snags = Snag.query.all()
    html_content = '<h1>Client Snags Report</h1>'
    for s in snags:
        html_content += f'<p><strong>{s.client_name}</strong>: {s.description}</p>'
        if s.image_path:
            html_content += f'<img src="{s.image_path}" width="200" />'
    pdf_path = 'snags_report.pdf'
    pdfkit.from_string(html_content, pdf_path)
    return send_file(pdf_path, as_attachment=True)

if __name__ == '__main__':
    db.create_all()
    app.run(debug=True)
