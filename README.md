import React, { useState, useEffect } from 'react';
import { Trash2, Plus, Beer, Send, AlertCircle, Clock, DollarSign, Phone, ChevronDown, ChevronUp, Settings } from 'lucide-react';

export default function App() {
  // Estado para la lista de clientes (con historial)
  const [clients, setClients] = useState(() => {
    const saved = localStorage.getItem('beerClientsV2');
    if (saved) {
      return JSON.parse(saved);
    }
    // Intento de migrar datos viejos si existen
    const oldSaved = localStorage.getItem('beerClients');
    if (oldSaved) {
      return JSON.parse(oldSaved).map(c => ({...c, phone: '', history: []}));
    }
    return [];
  });

  // Configuración global (Precio de la cerveza)
  const [beerPrice, setBeerPrice] = useState(() => {
    const saved = localStorage.getItem('beerPrice');
    return saved ? parseFloat(saved) : 50; // 50 Bs por defecto
  });

  // Estados para el formulario de nuevo cliente
  const [newClientName, setNewClientName] = useState('');
  const [newClientPhone, setNewClientPhone] = useState('');

  // Estado para los modales personalizados (Reemplazo de alert y confirm)
  const [modalInfo, setModalInfo] = useState({ isOpen: false, title: '', message: '', onConfirm: null, isAlert: true });

  // Guardar en localStorage cada vez que cambian los datos
  useEffect(() => {
    localStorage.setItem('beerClientsV2', JSON.stringify(clients));
  }, [clients]);

  useEffect(() => {
    localStorage.setItem('beerPrice', beerPrice.toString());
  }, [beerPrice]);

  const showAlert = (title, message) => {
    setModalInfo({ isOpen: true, title, message, onConfirm: null, isAlert: true });
  };

  const showConfirm = (title, message, onConfirm) => {
    setModalInfo({ isOpen: true, title, message, onConfirm, isAlert: false });
  };

  const closeModal = () => {
    setModalInfo({ isOpen: false, title: '', message: '', onConfirm: null, isAlert: true });
  };

  const addClient = (e) => {
    e.preventDefault();
    if (!newClientName.trim()) return;

    // Limpiar número (solo dejar números y el + para códigos de país)
    const cleanPhone = newClientPhone.replace(/[^\d+]/g, '');

    const newClient = {
      id: Date.now().toString(),
      name: newClientName.trim(),
      phone: cleanPhone,
      available: 0,
      consumed: 0,
      history: [] // Historial de acciones
    };

    setClients([newClient, ...clients]);
    setNewClientName('');
    setNewClientPhone('');
  };

  const removeClient = (id) => {
    showConfirm(
      'Eliminar Cliente',
      '¿Estás seguro de eliminar a este cliente? Se perderá todo su historial y registro.',
      () => {
        setClients(clients.filter(client => client.id !== id));
        closeModal();
      }
    );
  };

  const addBeers = (id, amount) => {
    const totalCost = amount * beerPrice;
    const now = new Date();
    
    setClients(clients.map(client => {
      if (client.id === id) {
        const historyEntry = {
          id: Date.now().toString(),
          type: 'COMPRA',
          amount: amount,
          totalBs: totalCost,
          date: now.toLocaleString('es-VE') // Formato local Venezuela
        };
        return { 
          ...client, 
          available: client.available + amount,
          history: [historyEntry, ...(client.history || [])] // Agregamos al inicio del historial
        };
      }
      return client;
    }));
  };

  const serveBeer = (id) => {
    setClients(clients.map(client => {
      if (client.id === id) {
        if (client.available > 0) {
          const historyEntry = {
            id: Date.now().toString(),
            type: 'ENTREGA',
            amount: 1,
            date: new Date().toLocaleString('es-VE')
          };
          return { 
            ...client, 
            available: client.available - 1,
            consumed: client.consumed + 1,
            history: [historyEntry, ...(client.history || [])]
          };
        } else {
          showAlert('Saldo Agotado', `¡${client.name} no tiene cervezas disponibles! Debe comprar más.`);
        }
      }
      return client;
    }));
  };

  const sendWhatsAppTicket = (client) => {
    // Calcular total pagado en la historia
    const totalPaid = (client.history || [])
        .filter(h => h.type === 'COMPRA')
        .reduce((sum, h) => sum + h.totalBs, 0);

    const totalBought = client.available + client.consumed;
    
    // Crear el mensaje con formato para WhatsApp
    const text = `🧾 *TICKET DE CONSUMO* 🍺\n\n` +
                 `👤 Cliente: *${client.name}*\n` +
                 `💰 Total Pagado: *Bs. ${totalPaid.toLocaleString('es-VE')}*\n\n` +
                 `✅ Total Cervezas Compradas: ${totalBought}\n` +
                 `🍺 Cervezas Consumidas: ${client.consumed}\n` +
                 `*🟢 DISPONIBLES: ${client.available}*\n\n` +
                 `¡Salud! 🍻`;
    
    const encodedText = encodeURIComponent(text);
    
    let url = '';
    // Si tiene número registrado, enviarlo directo, si no, ab
