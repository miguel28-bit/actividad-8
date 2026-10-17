# actividad-8
# Miguelmontealegre-Juandavidespitialeon

Simulador de red descentralizada básica con dos nodos independientes (Servidor A y Servidor B) en Python con Flask. Cada mensaje se empaqueta en un bloque conectado mediante SHA-256 y se sincroniza entre servidores.

---

## Código Principal (`app.py` / `main.py`)

```python
import hashlib
import time
import json
from flask import Flask, request, jsonify
import threading
import requests

# --- 1. CLASE BLOCKCHAIN (Con control de concurrencia y validación) ---
class Blockchain:
    def __init__(self):
        self.chain = []
        self.lock = threading.Lock()  # Cerrojo para evitar condiciones de carrera
        # Crear el bloque génesis (primer bloque)
        self.create_block(data="Bloque Genesis (Inicio)", previous_hash="0")

    def create_block(self, data, previous_hash):
        with self.lock:
            block = {
                'index': len(self.chain) + 1,
                'timestamp': time.time(),
                'data': data,
                'previous_hash': previous_hash
            }
            block['hash'] = self.hash_block(block)
            self.chain.append(block)
            return block

    def hash_block(self, block):
        # Genera la huella digital criptográfica (SHA-256) excluyendo la clave 'hash' si ya existe
        block_copy = block.copy()
        block_copy.pop('hash', None)
        block_string = json.dumps(block_copy, sort_keys=True).encode()
        return hashlib.sha256(block_string).hexdigest()

    def get_last_block(self):
        with self.lock:
            return self.chain[-1]

    def add_block_externally(self, block):
        with self.lock:
            last_block = self.chain[-1]
            
            # Validación criptográfica de la integridad del bloque
            calculated_hash = self.hash_block(block)
            if block['hash'] != calculated_hash:
                return False, "Hash del bloque inválido (datos alterados)"
            
            # Validación de enlace con el último bloque
            if last_block['hash'] == block['previous_hash']:
                self.chain.append(block)
                return True, "Bloque aceptado"
            
            return False, "El bloque no encaja con la cadena actual"


# --- 2. CONFIGURACIÓN DE LOS SERVIDORES (NODO A y NODO B) ---
app_a = Flask("Servidor_A")
app_b = Flask("Servidor_B")

blockchain_a = Blockchain()
blockchain_b = Blockchain()


# --- RUTAS DEL SERVIDOR A (Puerto 5000) ---
@app_a.route('/enviar', methods=['POST'])
def enviar_a():
    mensaje = request.json.get('mensaje')
    if not mensaje:
        return jsonify({"error": "Mensaje vacío"}), 400

    last_block = blockchain_a.get_last_block()
    nuevo_bloque = blockchain_a.create_block(mensaje, last_block['hash'])
    
    # Sincronización automática con el Servidor B
    try:
        response = requests.post('http://localhost:5001/recibir', json=nuevo_bloque, timeout=3)
        if response.status_code != 201:
            return jsonify({
                "status": "Guardado en A, pero B rechazó el bloque",
                "bloque": nuevo_bloque
            }), 201
    except requests.exceptions.RequestException:
        return jsonify({
            "status": "Guardado en A, pero B no estaba disponible para sincronizar",
            "bloque": nuevo_bloque
        }), 201
        
    return jsonify({
        "status": "Mensaje guardado en A y propagado a B",
