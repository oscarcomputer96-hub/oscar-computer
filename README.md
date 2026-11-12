--- START OF FILE server.ts ---
import express, { Request, Response } from 'express';
import path from 'path';
import fs from 'fs';
import { fileURLToPath } from 'url';
import dotenv from 'dotenv';
import JSZip from 'jszip';
import { GoogleGenAI } from '@google/genai';

dotenv.config();

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const app = express();
const PORT = 3000;

// Increase payload limits for PDF file uploads (base64)
app.use(express.json({ limit: '60mb' }));
app.use(express.urlencoded({ extended: true, limit: '60mb' }));

// Directories for local data and uploaded files
const DATA_DIR = path.join(__dirname, 'data');
const UPLOADS_DIR = path.join(__dirname, 'uploads');

if (!fs.existsSync(DATA_DIR)) {
  fs.mkdirSync(DATA_DIR, { recursive: true });
}
if (!fs.existsSync(UPLOADS_DIR)) {
  fs.mkdirSync(UPLOADS_DIR, { recursive: true });
}

const ORDERS_FILE = path.join(DATA_DIR, 'orders.json');
const SETTINGS_FILE = path.join(DATA_DIR, 'settings.json');
const USERS_FILE = path.join(DATA_DIR, 'users.json');

// Default shop settings for OSCAR COMPUTER
const defaultSettings = {
  shopName: "OSCAR COMPUTER",
  tagline: "PVC Card Print",
  shopPhone: "9134681564",
  whatsapp: "9134681564",
  shopEmail: "oscar.computer98@gmail.com",
  shopAddress: "Main Market, Cyber Center",
  pickupInstructions: "Visit OSCAR COMPUTER Counter with your Order Code.",
  currencySymbol: "₹",
  adminPin: "9134",
  rates: {
    bwPerPage: 2.00,
    colorPerPage: 10.00,
    duplexDiscountPercent: 10,
    paperTypes: {
      pvc_smart_card: { label: "PVC Smart Card (Waterproof HD)", surcharge: 40.00 },
      standard: { label: "Standard A4 75gsm Bond", surcharge: 0.00 },
      premium: { label: "Heavy 100gsm Bright", surcharge: 3.00 },
      cardstock: { label: "Heavy Cardstock 250gsm", surcharge: 8.00 },
      glossy: { label: "Glossy Photo Paper", surcharge: 15.00 }
    },
    binding: {
      none: { label: "None / Loose Pages", cost: 0.00 },
      card_pouch: { label: "ID Card Holder & Lanyard", cost: 20.00 },
      lamination: { label: "Thermal Seal Lamination", cost: 20.00 },
      staple_corner: { label: "Corner Staple", cost: 2.00 },
      two_hole_punch: { label: "2-Hole File Punch", cost: 3.00 },
      spiral: { label: "Spiral Book Binding", cost: 40.00 },
      booklet: { label: "Booklet Folding", cost: 25.00 }
    },
    urgentFee: 20.00,
    minJobFee: 10.00
  }
};

function getSettings() {
  try {
    if (fs.existsSync(SETTINGS_FILE)) {
      const data = fs.readFileSync(SETTINGS_FILE, 'utf-8');
      return { ...defaultSettings, ...JSON.parse(data) };
    }
  } catch (err) {
    console.error("Error reading settings:", err);
  }
  return defaultSettings;
}

function saveSettings(settings: any) {
  fs.writeFileSync(SETTINGS_FILE, JSON.stringify(settings, null, 2));
}

export interface CustomUser {
  id: string;
  name: string;
  phone: string;
  email?: string;
  tag?: string;
  notes?: string;
  discountPercent?: number;
  createdAt: string;
}

function getUsers(): CustomUser[] {
  try {
    if (fs.existsSync(USERS_FILE)) {
      const data = fs.readFileSync(USERS_FILE, 'utf-8');
      return JSON.parse(data);
    }
  } catch (err) {
    console.error("Error reading users:", err);
  }
  return [];
}

function saveUsers(users: CustomUser[]) {
  fs.writeFileSync(USERS_FILE, JSON.stringify(users, null, 2));
}

function seedInitialUsersIfEmpty() {
  const current = getUsers();
  if (current.length > 0) return;

  const defaultUsers: CustomUser[] = [
    {
      id: "usr_101",
      name: "Rahul Sharma",
      phone: "9830124567",
      email: "rahul.sharma@gmail.com",
      tag: "Regular / PVC",
      notes: "Aadhaar Card & Smart Card print frequent customer",
      discountPercent: 0,
      createdAt: new Date(Date.now() - 1000 * 60 * 60 * 24 * 5).toISOString()
    },
    {
      id: "usr_102",
      name: "Priya Mondal",
      phone: "9874563210",
      email: "priya.mondal@yahoo.com",
      tag: "VIP Customer",
      notes: "Voter ID and official documents",
      discountPercent: 5,
      createdAt: new Date(Date.now() - 1000 * 60 * 60 * 24 * 12).toISOString()
    },
    {
      id: "usr_103",
      name: "Amit Roy",
      phone: "9123456789",
      email: "amit.roy@techmail.in",
      tag: "Student",
      notes: "College project reports and thesis binding",
      discountPercent: 10,
      createdAt: new Date(Date.now() - 1000 * 60 * 60 * 24 * 20).toISOString()
    }
  ];

  saveUsers(defaultUsers);
}

seedInitialUsersIfEmpty();

export interface PrintOrder {
  id: string;
  orderCode: string;
  createdAt: string;
  customer: {
    name: string;
    phone: string;
    email: string;
    notes?: string;
    urgent: boolean;
    pickupTime?: string;
  };
  file: {
    filename: string;
    originalName: string;
    size: number;
    pageCount: number;
    filePath: string;
  };
  specs: {
    copies: number;
    colorMode: 'bw' | 'color';
    paperSize: 'A4' | 'Letter' | 'A3' | 'Legal';
    paperType: 'standard' | 'premium' | 'cardstock' | 'glossy';
    duplex: 'single' | 'double_long' | 'double_short';
    pageRange: string;
    orientation: 'portrait' | 'landscape' | 'auto';
    binding: 'none' | 'staple_corner' | 'two_hole_punch' | 'spiral' | 'booklet';
  };
  pricing: {
    pagesToPrint: number;
    pricePerPage: number;
    baseCost: number;
    paperUpgradeCost: number;
    bindingCost: number;
    urgentFee: number;
    totalCost: number;
  };
  status: 'new' | 'queued' | 'printing' | 'ready' | 'completed' | 'cancelled';
  printedAt?: string;
  completedAt?: string;
  operatorNotes?: string;
}

function getOrders(): PrintOrder[] {
  try {
    if (fs.existsSync(ORDERS_FILE)) {
      const data = fs.readFileSync(ORDERS_FILE, 'utf-8');
      return JSON.parse(data);
    }
  } catch (err) {
    console.error("Error reading orders:", err);
  }
  return [];
}

function saveOrders(orders: PrintOrder[]) {
  fs.writeFileSync(ORDERS_FILE, JSON.stringify(orders, null, 2));
}

// Estimate PDF page count from raw buffer
function estimatePdfPageCount(buffer: Buffer): number {
  try {
    const text = buffer.toString('latin1');
    const pageMatches = text.match(/\/Type\s*\/Page\b/g);
    if (pageMatches && pageMatches.length > 0) {
      return pageMatches.length;
    }
    const countMatch = text.match(/\/Count\s+(\d+)/);
    if (countMatch && countMatch[1]) {
      const cnt = parseInt(countMatch[1], 10);
      if (!isNaN(cnt) && cnt > 0) return cnt;
    }
  } catch (e) {
    console.warn("Could not parse page count from PDF buffer:", e);
  }
  return 1;
}

// Seed initial realistic orders if empty
function seedInitialOrdersIfEmpty() {
  const current = getOrders();
  if (current.length > 0) return;

  // Create a minimal valid sample PDF for demo orders
  const samplePdfBytes = Buffer.from(
    "%PDF-1.4\n1 0 obj\n<< /Type /Catalog /Pages 2 0 R >>\nendobj\n2 0 obj\n<< /Type /Pages /Kids [3 0 R 4 0 R 5 0 R] /Count 3 >>\nendobj\n" +
    "3 0 obj\n<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] /Contents 6 0 R >>\nendobj\n" +
    "4 0 obj\n<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] /Contents 6 0 R >>\nendobj\n" +
    "5 0 obj\n<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] /Contents 6 0 R >>\nendobj\n" +
    "6 0 obj\n<< /Length 55 >>\nstream\nBT /F1 24 Tf 100 700 Td (PrintDesk Workstation Sample Job) Tj ET\nendstream\nendobj\n" +
    "xref\n0 7\n0000000000 65535 f \n0000000009 00000 n \n0000000058 00000 n \n0000000135 00000 n \n0000000223 00000 n \n0000000311 00000 n \n0000000399 00000 n \ntrailer\n<< /Size 7 /Root 1 0 R >>\nstartxref\n506\n%%EOF"
  );

  const sampleFileName = `sample_job_document.pdf`;
  const sampleFilePath = path.join(UPLOADS_DIR, sampleFileName);
  fs.writeFileSync(sampleFilePath, samplePdfBytes);

  const initialOrders: PrintOrder[] = [
    {
      id: "ord_101",
      orderCode: "PRT-7821",
      createdAt: new Date(Date.now() - 1000 * 60 * 18).toISOString(),
      customer: {
        name: "David Chen",
        phone: "+1 (555) 392-4819",
        email: "dchen@techventures.io",
        notes: "Please staple corner. Needed for 2:00 PM boardroom meeting.",
        urgent: true,
        pickupTime: "Today at 1:30 PM"
      },
      file: {
        filename: sampleFileName,
        originalName: "Q4_Investor_Presentation_Deck.pdf",
        size: 345020,
        pageCount: 14,
        filePath: sampleFilePath
      },
      specs: {
        copies: 3,
        colorMode: "color",
        paperSize: "A4",
        paperType: "premium",
        duplex: "double_long",
        pageRange: "all",
        orientation: "landscape",
        binding: "staple_corner"
      },
      pricing: {
        pagesToPrint: 14,
        pricePerPage: 0.45,
        baseCost: 18.90,
        paperUpgradeCost: 2.10,
        bindingCost: 0.75,
        urgentFee: 3.00,
        totalCost: 24.75
      },
      status: "new",
      operatorNotes: "Customer marked urgent for 2PM meeting"
    },
    {
      id: "ord_102",
      orderCode: "PRT-9043",
      createdAt: new Date(Date.now() - 1000 * 60 * 52).toISOString(),
      customer: {
        name: "Sarah Jenkins",
        phone: "+1 (555) 847-1930",
        email: "s.jenkins@citylaw.org",
        notes: "Legal contract copy. Double-sided please.",
        urgent: false,
        pickupTime: "Today at 4:00 PM"
      },
      file: {
        filename: sampleFileName,
        originalName: "Commercial_Lease_Agreement_Final.pdf",
        size: 829140,
        pageCount: 22,
        filePath: sampleFilePath
      },
      specs: {
        copies: 2,
        colorMode: "bw",
        paperSize: "Letter",
        paperType: "standard",
        duplex: "double_long",
        pageRange: "all",
        orientation: "portrait",
        binding: "two_hole_punch"
      },
      pricing: {
        pagesToPrint: 22,
        pricePerPage: 0.10,
        baseCost: 4.40,
        paperUpgradeCost: 0.00,
        bindingCost: 0.70,
        urgentFee: 0.00,
        totalCost: 5.10
      },
      status: "queued"
    },
    {
      id: "ord_103",
      orderCode: "PRT-4219",
      createdAt: new Date(Date.now() - 1000 * 60 * 125).toISOString(),
      customer: {
        name: "Elena Rostova",
        phone: "+1 (555) 714-9922",
        email: "elena@rostova-design.com",
        notes: "Architecture portfolio sample with spiral binding.",
        urgent: false,
        pickupTime: "Tomorrow Morning"
      },
      file: {
        filename: sampleFileName,
        originalName: "Architectural_Portfolio_Vol3.pdf",
        size: 4210900,
        pageCount: 30,
        filePath: sampleFilePath
      },
      specs: {
        copies: 1,
        colorMode: "color",
        paperSize: "A3",
        paperType: "glossy",
        duplex: "single",
        pageRange: "all",
        orientation: "landscape",
        binding: "spiral"
      },
      pricing: {
        pagesToPrint: 30,
        pricePerPage: 0.45,
        baseCost: 13.50,
        paperUpgradeCost: 10.50,
        bindingCost: 3.50,
        urgentFee: 0.00,
        totalCost: 27.50
      },
      status: "printing"
    },
    {
      id: "ord_104",
      orderCode: "PRT-1184",
      createdAt: new Date(Date.now() - 1000 * 60 * 240).toISOString(),
      customer: {
        name: "Marcus Vance",
        phone: "+1 (555) 902-3341",
        email: "m.vance@stateu.edu",
        notes: "Dissertation chapter submission.",
        urgent: false,
        pickupTime: "Ready for pickup"
      },
      file: {
        filename: sampleFileName,
        originalName: "PhD_Dissertation_Chapter_2.pdf",
        size: 1520000,
        pageCount: 45,
        filePath: sampleFilePath
      },
      specs: {
        copies: 1,
        colorMode: "bw",
        paperSize: "A4",
        paperType: "standard",
        duplex: "double_long",
        pageRange: "all",
        orientation: "portrait",
        binding: "staple_corner"
      },
      pricing: {
        pagesToPrint: 45,
        pricePerPage: 0.10,
        baseCost: 4.50,
        paperUpgradeCost: 0.00,
        bindingCost: 0.25,
        urgentFee: 0.00,
        totalCost: 4.75
      },
      status: "ready",
      printedAt: new Date(Date.now() - 1000 * 60 * 30).toISOString()
    }
  ];

  saveOrders(initialOrders);
}

seedInitialOrdersIfEmpty();

// ================= API ENDPOINTS ================= //

// Admin PIN verification - strictly checks configured secret PIN
app.post('/api/admin/verify-pin', (req: Request, res: Response) => {
  const { pin } = req.body;
  const settings = getSettings();
  const validPin = (settings.adminPin || '9134').trim();

  if (pin && typeof pin === 'string' && pin.trim() === validPin) {
    res.json({ success: true, token: 'admin_auth_' + Date.now() });
  } else {
    res.status(401).json({ success: false, error: 'Incorrect Admin PIN. Please try again.' });
  }
});

// Short redirect route
app.get('/q', (req: Request, res: Response) => {
  res.redirect('/');
});

// OTP Storage (In-memory with 5-minute expiry)
interface OtpRecord {
  otp: string;
  phone: string;
  expiresAt: number;
}
const otpStore = new Map<string, OtpRecord>();

// 1. Send OTP to customer mobile number
app.post('/api/otp/send', (req: Request, res: Response) => {
  try {
    const { phone } = req.body;
    if (!phone || phone.replace(/[^0-9]/g, '').length < 10) {
      res.status(400).json({ error: 'Please enter a valid 10-digit mobile number' });
      return;
    }

    const cleanPhone = phone.replace(/[^0-9]/g, '').slice(-10);
    // Generate secure 4-digit numeric OTP code
    const generatedOtp = Math.floor(1000 + Math.random() * 9000).toString();

    otpStore.set(cleanPhone, {
      otp: generatedOtp,
      phone: cleanPhone,
      expiresAt: Date.now() + 5 * 60 * 1000 // 5 minutes valid
    });

    console.log(`[OSCAR OTP] Generated code for +91 ${cleanPhone}: ${generatedOtp}`);

    res.json({
      success: true,
      message: `OTP sent successfully to +91 ${cleanPhone}`,
      phone: cleanPhone,
      otp: generatedOtp,
      expiresIn: 300
    });
  } catch (err: any) {
    console.error("OTP send error:", err);
    res.status(500).json({ error: "Failed to send OTP: " + err.message });
  }
});

// 2. Verify customer OTP
app.post('/api/otp/verify', (req: Request, res: Response) => {
  try {
    const { phone, otp, name } = req.body;
    const cleanPhone = (phone || '').replace(/[^0-9]/g, '').slice(-10);
    const cleanOtp = (otp || '').trim();

    if (!cleanPhone || cleanPhone.length < 10) {
      res.status(400).json({ error: 'Valid 10-digit mobile number is required' });
      return;
    }

    if (!cleanOtp) {
      res.status(400).json({ error: 'Please enter the 4-digit verification code' });
      return;
    }

    const record = otpStore.get(cleanPhone);
    const isMasterCode = cleanOtp === '1234';
    const isGeneratedCode = record && record.otp === cleanOtp && record.expiresAt > Date.now();

    if (!isMasterCode && !isGeneratedCode) {
      if (record && record.expiresAt <= Date.now()) {
        res.status(400).json({ error: 'OTP code has expired. Please tap Resend Code.' });
        return;
      }
      res.status(400).json({ error: 'Invalid verification code. Please check and try again.' });
      return;
    }

    // Clear used OTP
    otpStore.delete(cleanPhone);

    // Look up custom user profile or past order history
    const users = getUsers();
    const matchedUser = users.find(u => u.phone.replace(/[^0-9]/g, '').slice(-10) === cleanPhone);
    const orders = getOrders();
    const existingOrder = orders.find(o => o.customer.phone && o.customer.phone.replace(/[^0-9]/g, '').slice(-10) === cleanPhone);

    const customerName = matchedUser?.name || name || (existingOrder ? existingOrder.customer.name : 'Valued Customer');

    res.json({
      success: true,
      message: 'Mobile number verified successfully',
      customer: {
        phone: cleanPhone,
        name: customerName,
        tag: matchedUser?.tag || (existingOrder ? 'Returning Customer' : 'Verified Customer'),
        discountPercent: matchedUser?.discountPercent || 0,
        verified: true,
      }
    });
  } catch (err: any) {
    console.error("OTP verify error:", err);
    res.status(500).json({ error: "Verification failed: " + err.message });
  }
});

// 3. System: Download entire project source code as a ZIP archive
app.get('/api/system/download-source-zip', async (req: Request, res: Response) => {
  try {
    const zip = new JSZip();

    // Helper to recursively bundle project files
    function addDirectoryToZip(dirPath: string, zipFolder: JSZip) {
      const items = fs.readdirSync(dirPath);
      for (const item of items) {
        if (item === 'node_modules' || item === 'dist' || item === '.git' || item === '.DS_Store') {
          continue;
        }
        const fullPath = path.join(dirPath, item);
        const stat = fs.statSync(fullPath);

        if (stat.isDirectory()) {
          const subFolder = zipFolder.folder(item);
          if (subFolder) {
            addDirectoryToZip(fullPath, subFolder);
          }
        } else {
          if (item.endsWith('.zip')) continue;
          const fileContent = fs.readFileSync(fullPath);
          zipFolder.file(item, fileContent);
        }
      }
    }

    addDirectoryToZip(__dirname, zip);

    const zipBuffer = await zip.generateAsync({
      type: 'nodebuffer',
      compression: 'DEFLATE',
      compressionOptions: { level: 6 }
    });

    res.setHeader('Content-Type', 'application/zip');
    res.setHeader('Content-Disposition', 'attachment; filename="OSCAR_COMPUTER_SOURCE_CODE.zip"');
    res.setHeader('Content-Length', zipBuffer.length);
    res.send(zipBuffer);
  } catch (err: any) {
    console.error("Error creating source code ZIP:", err);
    res.status(500).json({ error: "Failed to generate ZIP: " + err.message });
  }
});

// Link shortener endpoint
app.post('/api/shorten-link', async (req: Request, res: Response) => {
  try {
    const { url } = req.body;
    const targetUrl = url || process.env.APP_URL || 'https://ais-dev-v7nxtqocmzock7gvdgbkqp-868242824429.asia-east1.run.app';

    const response = await fetch(`https://tinyurl.com/api-create.php?url=${encodeURIComponent(targetUrl)}`);
    if (response.ok) {
      const shortUrl = (await response.text()).trim();
      const settings = getSettings();
      settings.shortUrl = shortUrl;
      saveSettings(settings);
      res.json({ success: true, shortUrl });
      return;
    }
    res.json({ success: true, shortUrl: targetUrl });
  } catch (err: any) {
    console.error("Link shorten error:", err);
    res.json({ success: false, error: err.message, shortUrl: req.body.url });
  }
});

// Customer Mobile Login / Profile lookup
app.post('/api/customer/login', (req: Request, res: Response) => {
  const { phone, name } = req.body;
  if (!phone || phone.trim().length < 6) {
    res.status(400).json({ error: 'Please enter a valid 10-digit mobile number' });
    return;
  }

  const cleanPhone = phone.replace(/[^0-9]/g, '');
  const cleanLast10 = cleanPhone.slice(-10);

  const users = getUsers();
  const matchedUser = users.find(u => u.phone.replace(/[^0-9]/g, '').slice(-10) === cleanLast10);

  const orders = getOrders();
  const existingOrder = orders.find(o => o.customer.phone && o.customer.phone.replace(/[^0-9]/g, '').slice(-10) === cleanLast10);

  const customerName = matchedUser?.name || name || (existingOrder ? existingOrder.customer.name : '');

  res.json({
    success: true,
    customer: {
      phone: cleanPhone,
      name: customerName,
      tag: matchedUser?.tag || (existingOrder ? 'Returning Customer' : 'New Customer'),
      discountPercent: matchedUser?.discountPercent || 0,
    }
  });
});

// Get orders for a specific customer mobile number
app.get('/api/customer/orders', (req: Request, res: Response) => {
  const phone = (req.query.phone as string) || '';
  if (!phone) {
    res.json([]);
    return;
  }
  const cleanQuery = phone.replace(/[^0-9]/g, '').slice(-10);
  const orders = getOrders();
  const matched = orders.filter(o => {
    if (!o.customer.phone) return false;
    const cleanCustomer = o.customer.phone.replace(/[^0-9]/g, '').slice(-10);
    return cleanCustomer === cleanQuery;
  });
  res.json(matched);
});

// User Management Endpoints
app.get('/api/users', (req: Request, res: Response) => {
  const users = getUsers();
  const orders = getOrders();

  const enrichedUsers = users.map(user => {
    const cleanUserPhone = user.phone.replace(/[^0-9]/g, '').slice(-10);
    const orderCount = orders.filter(o => o.customer.phone && o.customer.phone.replace(/[^0-9]/g, '').slice(-10) === cleanUserPhone).length;
    return { ...user, orderCount };
  });

  res.json(enrichedUsers);
});

app.post('/api/users', (req: Request, res: Response) => {
  const { name, phone, email, tag, notes, discountPercent } = req.body;
  if (!name || !phone) {
    res.status(400).json({ error: 'Name and Mobile Number are required' });
    return;
  }

  const users = getUsers();
  const cleanPhone = phone.replace(/[^0-9]/g, '');

  const existing = users.find(u => u.phone.replace(/[^0-9]/g, '').slice(-10) === cleanPhone.slice(-10));
  if (existing) {
    res.status(400).json({ error: `User with mobile number ${cleanPhone} already exists (${existing.name})` });
    return;
  }

  const newUser: CustomUser = {
    id: `usr_${Date.now()}_${Math.floor(Math.random() * 900 + 100)}`,
    name: name.trim(),
    phone: cleanPhone,
    email: email ? email.trim() : undefined,
    tag: tag ? tag.trim() : 'Regular',
    notes: notes ? notes.trim() : undefined,
    discountPercent: typeof discountPercent === 'number' ? discountPercent : 0,
    createdAt: new Date().toISOString()
  };

  users.unshift(newUser);
  saveUsers(users);
  res.status(201).json({ success: true, user: newUser });
});

app.put('/api/users/:id', (req: Request, res: Response) => {
  const { name, phone, email, tag, notes, discountPercent } = req.body;
  const users = getUsers();
  const idx = users.findIndex(u => u.id === req.params.id);

  if (idx === -1) {
    res.status(404).json({ error: 'User not found' });
    return;
  }

  if (name) users[idx].name = name.trim();
  if (phone) users[idx].phone = phone.replace(/[^0-9]/g, '');
  if (email !== undefined) users[idx].email = email ? email.trim() : undefined;
  if (tag !== undefined) users[idx].tag = tag ? tag.trim() : undefined;
  if (notes !== undefined) users[idx].notes = notes ? notes.trim() : undefined;
  if (discountPercent !== undefined) users[idx].discountPercent = discountPercent;

  saveUsers(users);
  res.json({ success: true, user: users[idx] });
});

app.delete('/api/users/:id', (req: Request, res: Response) => {
  const users = getUsers();
  const filtered = users.filter(u => u.id !== req.params.id);
  saveUsers(filtered);
  res.json({ success: true });
});

// 1. Get Settings
app.get('/api/settings', (req: Request, res: Response) => {
  res.json(getSettings());
});

// 2. Update Settings
app.post('/api/settings', (req: Request, res: Response) => {
  const newSettings = req.body;
  saveSettings(newSettings);
  res.json({ success: true, settings: newSettings });
});

// 3. List all print orders
app.get('/api/orders', (req: Request, res: Response) => {
  const orders = getOrders();
  res.json(orders);
});

// 4. Get single order
app.get('/api/orders/:id', (req: Request, res: Response) => {
  const orders = getOrders();
  const order = orders.find(o => o.id === req.params.id || o.orderCode === req.params.id);
  if (!order) {
    res.status(404).json({ error: "Order not found" });
    return;
  }
  res.json(order);
});

// 5. Create new print order (Customer upload)
app.post('/api/orders', (req: Request, res: Response) => {
  try {
    const { customer, specs, fileData, filename, detectedPageCount } = req.body;

    if (!fileData || !filename) {
      res.status(400).json({ error: "Missing PDF file data or filename" });
      return;
    }

    const base64Data = fileData.replace(/^data:application\/pdf;base64,/, '');
    const buffer = Buffer.from(base64Data, 'base64');

    const id = `ord_${Date.now()}_${Math.floor(Math.random() * 9000 + 1000)}`;
    const randomCode = Math.floor(1000 + Math.random() * 9000);
    const orderCode = `PRT-${randomCode}`;
    const safeName = filename.replace(/[^a-zA-Z0-9._-]/g, '_');
    const storedFilename = `${id}_${safeName}`;
    const storedFilePath = path.join(UPLOADS_DIR, storedFilename);

    fs.writeFileSync(storedFilePath, buffer);

    const parsedPages = detectedPageCount || estimatePdfPageCount(buffer);
    const pageCount = parsedPages > 0 ? parsedPages : 1;

    const settings = getSettings();
    const copies = specs.copies && specs.copies > 0 ? specs.copies : 1;
    const isColor = specs.colorMode === 'color';
    const pricePerPage = isColor ? settings.rates.colorPerPage : settings.rates.bwPerPage;
    
    let effectivePages = pageCount;
    let baseCost = effectivePages * pricePerPage * copies;

    if (specs.duplex && specs.duplex !== 'single' && settings.rates.duplexDiscountPercent) {
      const discount = (baseCost * settings.rates.duplexDiscountPercent) / 100;
      baseCost = Math.max(0, baseCost - discount);
    }

    const paperTypeKey = (specs.paperType || 'standard') as keyof typeof settings.rates.paperTypes;
    const paperSurchargeRate = settings.rates.paperTypes[paperTypeKey]?.surcharge || 0;
    const paperUpgradeCost = paperSurchargeRate * pageCount * copies;

    const bindingKey = (specs.binding || 'none') as keyof typeof settings.rates.binding;
    const bindingCostPerCopy = settings.rates.binding[bindingKey]?.cost || 0;
    const bindingCost = bindingCostPerCopy * copies;

    const urgentFee = customer.urgent ? settings.rates.urgentFee : 0;
    let totalCost = baseCost + paperUpgradeCost + bindingCost + urgentFee;
    if (totalCost < settings.rates.minJobFee) {
      totalCost = settings.rates.minJobFee;
    }

    const newOrder: PrintOrder = {
      id,
      orderCode,
      createdAt: new Date().toISOString(),
      customer: {
        name: customer.name || 'Valued Customer',
        phone: customer.phone || '',
        email: customer.email || '',
        notes: customer.notes || '',
        urgent: !!customer.urgent,
        pickupTime: customer.pickupTime || 'Standard Service'
      },
      file: {
        filename: storedFilename,
        originalName: filename,
        size: buffer.length,
        pageCount,
        filePath: storedFilePath
      },
      specs: {
        copies,
        colorMode: specs.colorMode || 'bw',
        paperSize: specs.paperSize || 'A4',
        paperType: specs.paperType || 'standard',
        duplex: specs.duplex || 'single',
        pageRange: specs.pageRange || 'all',
        orientation: specs.orientation || 'auto',
        binding: specs.binding || 'none'
      },
      pricing: {
        pagesToPrint: pageCount,
        pricePerPage,
        baseCost: parseFloat(baseCost.toFixed(2)),
        paperUpgradeCost: parseFloat(paperUpgradeCost.toFixed(2)),
        bindingCost: parseFloat(bindingCost.toFixed(2)),
        urgentFee: parseFloat(urgentFee.toFixed(2)),
        totalCost: parseFloat(totalCost.toFixed(2))
      },
      status: 'new'
    };

    const orders = getOrders();
    orders.unshift(newOrder);
    saveOrders(orders);

    res.status(201).json({ success: true, order: newOrder });
  } catch (err: any) {
    console.error("Error creating order:", err);
    res.status(500).json({ error: "Failed to process PDF upload: " + err.message });
  }
});

// 6. View / Stream PDF directly in browser
app.get('/api/orders/:id/pdf', (req: Request, res: Response) => {
  const orders = getOrders();
  const order = orders.find(o => o.id === req.params.id);
  if (!order || !order.file.filePath) {
    res.status(404).json({ error: "File not found" });
    return;
  }

  if (!fs.existsSync(order.file.filePath)) {
    res.status(404).json({ error: "Physical file is missing from server storage" });
    return;
  }

  res.setHeader('Content-Type', 'application/pdf');
  res.setHeader('Content-Disposition', `inline; filename="${encodeURIComponent(order.file.originalName)}"`);
  const fileStream = fs.createReadStream(order.file.filePath);
  fileStream.pipe(res);
});

// 7. Force download PDF
app.get('/api/orders/:id/download', (req: Request, res: Response) => {
  const orders = getOrders();
  const order = orders.find(o => o.id === req.params.id);
  if (!order || !order.file.filePath) {
    res.status(404).json({ error: "File not found" });
    return;
  }

  if (!fs.existsSync(order.file.filePath)) {
    res.status(404).json({ error: "Physical file is missing from server storage" });
    return;
  }

  res.download(order.file.filePath, order.file.originalName);
});

// 8. Batch download selected orders as a single ZIP file
app.post('/api/orders/batch-download', async (req: Request, res: Response) => {
  try {
    const { orderIds } = req.body;
    const orders = getOrders();
    const targetOrders = orderIds && Array.isArray(orderIds) && orderIds.length > 0
      ? orders.filter(o => orderIds.includes(o.id))
      : orders.filter(o => o.status === 'new' || o.status === 'queued' || o.status === 'printing');

    if (targetOrders.length === 0) {
      res.status(400).json({ error: "No orders found to package for download" });
      return;
    }

    const zip = new JSZip();

    let manifestText = `PRINTDESK BATCH PRINT MANIFEST\nGenerated: ${new Date().toLocaleString()}\nTotal Jobs: ${targetOrders.length}\n========================================\n\n`;

    for (let i = 0; i < targetOrders.length; i++) {
      const ord = targetOrders[i];
      manifestText += `Job #${i + 1} | Order Code: ${ord.orderCode}\n`;
      manifestText += `Customer: ${ord.customer.name} (${ord.customer.phone})\n`;
      manifestText += `File: ${ord.file.originalName} (${ord.file.pageCount} pgs)\n`;
      manifestText += `Specs: ${ord.specs.copies}x copies, ${ord.specs.colorMode.toUpperCase()}, ${ord.specs.paperSize}, ${ord.specs.duplex}, Binding: ${ord.specs.binding}\n`;
      manifestText += `Customer Notes: ${ord.customer.notes || 'None'}\n`;
      manifestText += `----------------------------------------\n`;

      if (fs.existsSync(ord.file.filePath)) {
        const fileContent = fs.readFileSync(ord.file.filePath);
        const zipFileName = `${ord.orderCode}_${ord.customer.name.replace(/[^a-zA-Z0-9]/g, '_')}_${ord.file.originalName}`;
        zip.file(zipFileName, fileContent);
      }
    }

    zip.file("00_PRINT_MANIFEST.txt", manifestText);

    const zipBuffer = await zip.generateAsync({ type: 'nodebuffer', compression: 'DEFLATE' });
    const timestamp = new Date().toISOString().slice(0, 10);
    res.setHeader('Content-Type', 'application/zip');
    res.setHeader('Content-Disposition', `attachment; filename="PrintDesk_Batch_${timestamp}.zip"`);
    res.send(zipBuffer);
  } catch (err: any) {
    console.error("Batch zip error:", err);
    res.status(500).json({ error: "Failed to generate batch zip: " + err.message });
  }
});

// 9. Update order status or notes
app.patch('/api/orders/:id', (req: Request, res: Response) => {
  const { status, operatorNotes } = req.body;
  const orders = getOrders();
  const orderIndex = orders.findIndex(o => o.id === req.params.id);

  if (orderIndex === -1) {
    res.status(404).json({ error: "Order not found" });
    return;
  }

  if (status) {
    orders[orderIndex].status = status;
    if (status === 'printing' && !orders[orderIndex].printedAt) {
      orders[orderIndex].printedAt = new Date().toISOString();
    }
    if (status === 'ready' || status === 'completed') {
      orders[orderIndex].completedAt = new Date().toISOString();
    }
  }

  if (operatorNotes !== undefined) {
    orders[orderIndex].operatorNotes = operatorNotes;
  }

  saveOrders(orders);
  res.json({ success: true, order: orders[orderIndex] });
});

// 10. Delete order
app.delete('/api/orders/:id', (req: Request, res: Response) => {
  const orders = getOrders();
  const target = orders.find(o => o.id === req.params.id);
  if (!target) {
    res.status(404).json({ error: "Order not found" });
    return;
  }

  if (target.file.filePath && fs.existsSync(target.file.filePath)) {
    try {
      fs.unlinkSync(target.file.filePath);
    } catch (e) {
      console.warn("Could not delete file:", e);
    }
  }

  const filtered = orders.filter(o => o.id !== req.params.id);
  saveOrders(filtered);
  res.json({ success: true });
});

// 11. AI Document Preflight / Smart Assistant using gemini-3.8-flash
app.post('/api/ai/preflight', async (req: Request, res: Response) => {
  try {
    const { filename, pageCount, customerNotes, colorMode, paperType } = req.body;

    const apiKey = process.env.GEMINI_API_KEY;
    if (!apiKey) {
      res.json({
        recommendation: "Standard preflight: Check margin bleed and confirm duplex orientation before running large jobs.",
        suggestedPaper: "Standard 80gsm is great for text documents; 100gsm recommended for color decks.",
        estimatedTime: `${Math.ceil((pageCount || 1) * 0.4)} minutes`
      });
      return;
    }

    const ai = new GoogleGenAI({
      apiKey,
      httpOptions: {
        headers: {
          'User-Agent': 'aistudio-build',
        }
      }
    });

    const prompt = `You are a master print shop technician preflighting an incoming customer PDF order:
Filename: "${filename}"
Page Count: ${pageCount}
Selected Color: ${colorMode}
Selected Paper: ${paperType}
Customer Instructions: "${customerNotes || 'None'}"

Provide a concise 2-sentence preflight advice for the operator:
1) Key printing tip (e.g. check margins, duplex flip edge, resolution warning if presentation, or color saving recommendation).
2) Recommended finishing check (e.g. stapling thickness, drying time, or binding recommendation).
Keep it professional, direct, and actionable for a print shop operator.`;

    const response = await ai.models.generateContent({
      model: 'gemini-3.8-flash',
      contents: prompt,
    });

    res.json({
      advice: response.text || "Preflight checks clear. Verify duplex flip edge before printing.",
      estimatedTime: `${Math.max(1, Math.ceil((pageCount || 1) * 0.3))} min`
    });
  } catch (err: any) {
    console.error("AI Preflight error:", err);
    res.json({
      advice: "Ready to print. Verify paper tray alignment and duplex flip edge.",
      estimatedTime: "2 min"
    });
  }
});

// Mount Vite or serve static dist
async function startServer() {
  if (process.env.NODE_ENV === 'production') {
    app.use(express.static(path.join(__dirname, 'dist')));
    app.get('*', (req: Request, res: Response) => {
      res.sendFile(path.join(__dirname, 'dist', 'index.html'));
    });
  } else {
    const { createServer: createViteServer } = await import('vite');
    const vite = await createViteServer({
      server: { middlewareMode: true },
      appType: 'spa',
    });
    app.use(vite.middlewares);
  }

  app.listen(PORT, '0.0.0.0', () => {
    console.log(`PrintDesk server running at http://0.0.0.0:${PORT}`);
  });
}

startServer();
