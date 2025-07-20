import React, { createContext, useContext, useState, useEffect } from "react";
import { auth, db } from "@/lib/firebase";
import {
  onAuthStateChanged,
  createUserWithEmailAndPassword,
  signInWithEmailAndPassword,
} from "firebase/auth";
import {
  collection,
  getDocs,
  addDoc,
  updateDoc,
  deleteDoc,
  doc,
  query,
  where,
} from "firebase/firestore";
import { useRouter } from "next/router";
import { Button } from "@/components/ui/button";
import {
  Card,
  CardContent,
  CardHeader,
  CardTitle,
} from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Badge } from "@/components/ui/badge";
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { Sheet, SheetTrigger, SheetContent } from "@/components/ui/sheet";
import { Separator } from "@/components/ui/separator";
import jsPDF from "jspdf";
import autoTable from "jspdf-autotable";
import emailjs from "emailjs-com";
import { loadStripe } from "@stripe/stripe-js";

const stripePromise = loadStripe("pk_test_REPLACE_WITH_YOUR_PUBLIC_KEY");

const CartContext = createContext();

export const useCart = () => useContext(CartContext);

export const CartProvider = ({ children }) => {
  const [cart, setCart] = useState([]);
  const [user, setUser] = useState(null);
  const [products, setProducts] = useState([]);
  const [orders, setOrders] = useState([]);
  const [allOrders, setAllOrders] = useState([]);
  const [orderFilter, setOrderFilter] = useState("");
  const router = useRouter();

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (user) => {
      if (user) {
        setUser(user);
        fetchOrders(user.uid);
        if (user.email === "admin@zestflix.com") {
          fetchAllOrders();
        }
      } else {
        setUser(null);
        setOrders([]);
        setAllOrders([]);
      }
    });
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    const fetchProducts = async () => {
      const querySnapshot = await getDocs(collection(db, "products"));
      const items = [];
      querySnapshot.forEach((doc) => {
        items.push({ id: doc.id, ...doc.data() });
      });
      setProducts(items);
    };
    fetchProducts();
  }, []);

  const addToCart = (product) => {
    setCart((prevCart) => [...prevCart, product]);
  };

  const removeFromCart = (id) => {
    setCart((prevCart) => prevCart.filter((item) => item.id !== id));
  };

  const clearCart = () => {
    setCart([]);
  };

  const isAdmin = user?.email === "admin@zestflix.com";

  const redirectIfNotAdmin = () => {
    if (!isAdmin) {
      router.push("/");
    }
  };

  const signup = async (email, password) => {
    await createUserWithEmailAndPassword(auth, email, password);
  };

  const login = async (email, password) => {
    await signInWithEmailAndPassword(auth, email, password);
  };

  const addProduct = async (product) => {
    await addDoc(collection(db, "products"), product);
  };

  const updateProduct = async (id, data) => {
    await updateDoc(doc(db, "products", id), data);
    setProducts((prev) => prev.map((p) => (p.id === id ? { ...p, ...data } : p)));
  };

  const deleteProduct = async (id) => {
    await deleteDoc(doc(db, "products", id));
    setProducts((prev) => prev.filter((p) => p.id !== id));
  };

  const updateOrderStatus = async (id, status) => {
    await updateDoc(doc(db, "orders", id), { status });
    setOrders((prev) => prev.map((o) => (o.id === id ? { ...o, status } : o)));
    setAllOrders((prev) => prev.map((o) => (o.id === id ? { ...o, status } : o)));
  };

  const sendEmailConfirmation = (order) => {
    const templateParams = {
      to_email: user.email,
      order_id: order.id,
      message: `Your order with ID ${order.id} has been placed successfully.`,
    };

    emailjs.send("service_id", "template_id", templateParams, "user_id").then(
      (result) => {
        console.log("Email sent", result.text);
      },
      (error) => {
        console.error("Email error", error.text);
      }
    );
  };

  const placeOrder = async () => {
    if (!user) return;

    const stripe = await stripePromise;
    const response = await fetch("/api/create-checkout-session", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ cart, userId: user.uid }),
    });

    const session = await response.json();
    const result = await stripe.redirectToCheckout({ sessionId: session.id });

    if (result.error) {
      console.error(result.error.message);
    }
  };

  const fetchOrders = async (userId) => {
    const q = query(collection(db, "orders"), where("userId", "==", userId));
    const querySnapshot = await getDocs(q);
    const ordersList = [];
    querySnapshot.forEach((doc) => {
      ordersList.push({ id: doc.id, ...doc.data() });
    });
    setOrders(ordersList);
  };

  const fetchAllOrders = async () => {
    const querySnapshot = await getDocs(collection(db, "orders"));
    const ordersList = [];
    querySnapshot.forEach((doc) => {
      ordersList.push({ id: doc.id, ...doc.data() });
    });
    setAllOrders(ordersList);
  };

  const filteredOrders = orderFilter
    ? allOrders.filter((order) => order.status.toLowerCase().includes(orderFilter.toLowerCase()))
    : allOrders;

  return (
    <CartContext.Provider
      value={{
        cart,
        user,
        products,
        orders,
        allOrders,
        filteredOrders,
        orderFilter,
        setOrderFilter,
        addToCart,
        removeFromCart,
        clearCart,
        isAdmin,
        redirectIfNotAdmin,
        signup,
        login,
        addProduct,
        updateProduct,
        deleteProduct,
        updateOrderStatus,
        placeOrder,
      }}
    >
      {children}
    </CartContext.Provider>
  );
};

export const OrderHistory = () => {
  const { orders } = useCart();

  const downloadInvoice = (order) => {
    const doc = new jsPDF();
    doc.text(`Invoice for Order ID: ${order.id}`, 10, 10);
    autoTable(doc, {
      head: [["Product", "Price"]],
      body: order.items.map((item) => [item.name, `₹${item.price}`]),
    });
    doc.text(
      `Date: ${new Date(order.createdAt).toLocaleString()}`,
      10,
      doc.autoTable.previous.finalY + 10
    );
    doc.save(`invoice_${order.id}.pdf`);
  };

  return (
    <div className="p-6">
      <h2 className="text-xl font-bold mb-4">Order History</h2>
      {orders.map((order) => (
        <Card key={order.id} className="mb-4">
          <CardHeader>
            <CardTitle>Order #{order.id}</CardTitle>
          </CardHeader>
          <CardContent>
            {order.items.map((item, idx) => (
              <div key={idx} className="flex justify-between">
                <span>{item.name}</span>
                <span>₹{item.price}</span>
              </div>
            ))}
            <p className="text-sm text-gray-500 mt-2">
              Date: {new Date(order.createdAt).toLocaleString()}
            </p>
            <p className="text-sm text-blue-500">Status: {order.status}</p>
            <Button className="mt-2" onClick={() => downloadInvoice(order)}>
              Download Invoice
            </Button>
          </CardContent>
        </Card>
      ))}
    </div>
  );
};
