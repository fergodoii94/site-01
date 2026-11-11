# site-01


'use client';

import React, { useState } from 'react';
import Script from 'next/script';

// ==========================================
// 1. CONFIGURAÇÕES E TIPAGENS
// ==========================================

// Substitua pelo seu ID real do Meta Pixel (ou use process.env.NEXT_PUBLIC_FACEBOOK_PIXEL_ID)
const PIXEL_ID = process.env.NEXT_PUBLIC_FACEBOOK_PIXEL_ID || '1234567890123456';

// Extensão global do Window para TypeScript reconhecer a função fbq do Meta Pixel
declare global {
  interface Window {
    fbq?: (...args: any[]) => void;
  }
}

export interface CartItem {
  id: string;
  name: string;
  quantity: number;
  price: number;
  variant?: string;
}

export interface CheckoutData {
  phone: string;
  items: CartItem[];
  customerName?: string;
  notes?: string;
}

// ==========================================
// 2. HELPERS (PIXEL & WHATSAPP)
// ==========================================

/**
 * Dispara eventos de conversão no Meta Pixel
 */
export const trackPixelEvent = (eventName: string, options: Record<string, any> = {}): void => {
  if (typeof window !== 'undefined' && window.fbq) {
    window.fbq('track', eventName, options);
  }
};

/**
 * Formata os itens do carrinho e dispara a mensagem no WhatsApp Web/App
 */
export function sendCartToWhatsApp({ phone, items, customerName, notes }: CheckoutData): void {
  const total = items.reduce((acc, item) => acc + item.price * item.quantity, 0);

  let message = `*Novo Pedido* 🛍️\n`;
  if (customerName) message += `*Cliente:* ${customerName}\n`;
  message += `----------------------------\n`;

  items.forEach((item) => {
    const variantText = item.variant ? ` (${item.variant})` : '';
    const itemTotal = (item.price * item.quantity).toFixed(2);
    message += `• ${item.quantity}x ${item.name}${variantText} - R$ ${itemTotal}\n`;
  });

  message += `----------------------------\n`;
  message += `*Total:* R$ ${total.toFixed(2)}\n`;
  if (notes) message += `\n*Observações:* ${notes}\n`;

  const encodedMessage = encodeURIComponent(message);
  const cleanPhone = phone.replace(/\D/g, '');
  const whatsappUrl = `https://wa.me/${cleanPhone}?text=${encodedMessage}`;

  window.open(whatsappUrl, '_blank', 'noopener,noreferrer');
}

// ==========================================
// 3. COMPONENTE DE INJEÇÃO DO META PIXEL
// ==========================================

export function MetaPixelScript() {
  if (!PIXEL_ID) return null;

  return (
    <Script
      id="fb-pixel-inline"
      strategy="afterInteractive"
      dangerouslySetInnerHTML={{
        __html: `
          !function(f,b,e,v,n,t,s)
          {if(f.fbq)return;n=f.fbq=function(){n.callMethod?
          n.callMethod.apply(n,arguments):n.queue.push(arguments)};
          if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
          n.queue=[];t=b.createElement(e);t.async=!0;
          t.src=v;s=b.getElementsByTagName(e)[0];
          s.parentNode.insertBefore(t,s)}(window, document,'script',
          'https://connect.facebook.net/en_US/fbevents.js');
          fbq('init', '${PIXEL_ID}');
          fbq('track', 'PageView');
        `,
      }}
    />
  );
}

// ==========================================
// 4. COMPONENTE PRINCIPAL DE CHECKOUT
// ==========================================

export default function CartCheckout() {
  const [customerName, setCustomerName] = useState('');
  const [notes, setNotes] = useState('');

  // Exemplo de lista de produtos presentes no carrinho
  const cartItems: CartItem[] = [
    { id: '1', name: 'Maryan Lash Volume Russo', quantity: 2, price: 45.90, variant: '0.07 C' },
    { id: '2', name: 'Pinça de Precisão Curva', quantity: 1, price: 32.00 },
  ];

  const handleCheckout = (e: React.FormEvent) => {
    e.preventDefault();
    if (cartItems.length === 0) return;

    const totalValue = cartItems.reduce((acc, item) => acc + item.price * item.quantity, 0);

    // 🎯 1. Rastreia a conversão no Meta Pixel
    trackPixelEvent('InitiateCheckout', {
      content_name: 'Carrinho WhatsApp',
      content_type: 'product',
      value: totalValue,
      currency: 'BRL',
      num_items: cartItems.length,
    });

    // 🎯 2. Dispara o pedido para o WhatsApp da loja
    sendCartToWhatsApp({
      phone: '34600000000', // Altere para o seu número comercial
      items: cartItems,
      customerName: customerName.trim(),
      notes: notes.trim(),
    });
  };

  return (
    <>
      {/* Script do Meta Pixel injetado dinamicamente */}
      <MetaPixelScript />

      {/* Card de Checkout em Tailwind CSS */}
      <div className="p-6 max-w-md mx-auto bg-white rounded-xl shadow-md border border-gray-100 font-sans">
        <h2 className="text-xl font-bold text-gray-800 mb-4">Finalizar Pedido</h2>

        {/* Resumo do Pedido */}
        <div className="mb-4 bg-gray-50 p-3 rounded-lg border border-gray-200">
          <h3 className="text-sm font-semibold text-gray-600 mb-2">Itens do Pedido:</h3>
          <ul className="space-y-1 text-sm text-gray-700">
            {cartItems.map((item) => (
              <li key={item.id} className="flex justify-between">
                <span>
                  {item.quantity}x {item.name} {item.variant ? `(${item.variant})` : ''}
                </span>
                <span className="font-medium">R$ {(item.price * item.quantity).toFixed(2)}</span>
              </li>
            ))}
          </ul>
        </div>

        {/* Form para dados do cliente */}
        <form onSubmit={handleCheckout} className="space-y-4">
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">Seu Nome</label>
            <input
              type="text"
              required
              value={customerName}
              onChange={(e) => setCustomerName(e.target.value)}
              placeholder="Digite seu nome completo"
              className="w-full border border-gray-300 rounded-lg p-2.5 text-sm focus:ring-2 focus:ring-green-500 focus:outline-none"
            />
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">
              Observações (opcional)
            </label>
            <textarea
              rows={2}
              value={notes}
              onChange={(e) => setNotes(e.target.value)}
              placeholder="Ex: Entregar após as 14h"
              className="w-full border border-gray-300 rounded-lg p-2.5 text-sm focus:ring-2 focus:ring-green-500 focus:outline-none"
            />
          </div>

          <button
            type="submit"
            className="w-full bg-green-600 hover:bg-green-700 text-white font-semibold py-3 px-4 rounded-lg transition duration-200 flex items-center justify-center gap-2 shadow-sm"
          >
            <span>Enviar Pedido via WhatsApp</span>
          </button>
        </form>
      </div>
    </>
  );
}
