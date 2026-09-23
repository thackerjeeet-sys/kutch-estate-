/**
 * Kutch Estate Link — Google Apps Script backend
 *
 * Files:
 *   Code.gs
 *   Untitled.html
 *   Admin.html
 */

const ADMIN_NAME = "jeet";
const ADMIN_PASSWORD = "192004";

const DB_FILE_NAME = "kutch-estate-link-database.json";
const TOKEN_TTL_MS = 6 * 60 * 60 * 1000; // 6 hours

function doGet(e) {
  const page = e && e.parameter && e.parameter.page;

  if (page === "admin") {
    return HtmlService
      .createHtmlOutputFromFile("Admin")
      .setTitle("Kutch Estate Link — Private Backend")
      .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
  }

  return HtmlService
    .createHtmlOutputFromFile("Untitled")
    .setTitle("કચ્છ એસ્ટેટ લિંક")
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

function doPost(e) {
  try {
    const body = JSON.parse(e.postData.contents || "{}");
    const action = body.action || "";

    if (action === "login") {
      return json_(login_(body.name, body.password));
    }

    if (!verifyToken_(body.token)) {
      return json_({
        ok: false,
        error: "Admin login required."
      });
    }

    switch (action) {

      case "adminData":
        return json_({
          ok: true,
          ...readDb_()
        });

      case "confirmPayment":
        return json_(
          confirmPayment_(body.paymentId)
        );

      case "cancelPayment":
        return json_(
          cancelPayment_(body.paymentId)
        );

      case "updateListing":
        return json_(
          updateListing_(body.listing)
        );

      case "deleteListing":
        return json_(
          deleteListing_(body.listingId)
        );

      case "updateRequirement":
        return json_(
          updateRequirement_(body.req)
        );

      case "deleteRequirement":
        return json_(
          deleteRequirement_(body.reqId)
        );

      default:
        return json_({
          ok: false,
          error: "Unknown action: " + action
        });
    }

  } catch (err) {
    return json_({
      ok: false,
      error: String(err && err.message || err)
    });
  }
}


/* =========================
   LOGIN
========================= */

function login_(name, password) {

  if (
    String(name || "") !== ADMIN_NAME ||
    String(password || "") !== ADMIN_PASSWORD
  ) {
    return {
      ok: false,
      error: "Invalid admin name or password."
    };
  }

  const token = makeToken_();

  return {
    ok: true,
    token: token
  };
}


/* =========================
   DATABASE
========================= */

function emptyDb_() {
  return {
    listings: [],
    pendingPayments: [],
    reqs: [],
    users: [],
    updatedAt: new Date().toISOString()
  };
}

function getDbFile_() {

  const files = DriveApp
    .getFilesByName(DB_FILE_NAME);

  if (files.hasNext()) {
    return files.next();
  }

  const db = emptyDb_();

  return DriveApp.createFile(
    DB_FILE_NAME,
    JSON.stringify(db, null, 2),
    MimeType.PLAIN_TEXT
  );
}

function readDb_() {

  const file = getDbFile_();

  let db;

  try {
    db = JSON.parse(
      file.getBlob().getDataAsString()
    );
  } catch (err) {
    db = emptyDb_();
  }

  db.listings = Array.isArray(db.listings)
    ? db.listings
    : [];

  db.pendingPayments =
    Array.isArray(db.pendingPayments)
      ? db.pendingPayments
      : [];

  db.reqs =
    Array.isArray(db.reqs)
      ? db.reqs
      : [];

  db.users =
    Array.isArray(db.users)
      ? db.users
      : [];

  return db;
}

function writeDb_(db) {

  db.updatedAt =
    new Date().toISOString();

  const file = getDbFile_();

  file.setContent(
    JSON.stringify(db, null, 2)
  );

  return db;
}


/* =========================
   PAYMENT CONFIRMATION
========================= */

function confirmPayment_(paymentId) {

  const db = readDb_();

  const payment =
    db.pendingPayments.find(
      x => String(x.id) === String(paymentId)
    );

  if (!payment) {
    return {
      ok: false,
      error: "Payment notification not found."
    };
  }

  payment.status = "confirmed";

  payment.confirmedAt =
    new Date().toISOString();

  if (payment.listing) {

    const listing = {
      ...payment.listing,

      status: "live",

      paymentStatus: "confirmed",

      publishedAt:
        new Date().toISOString()
    };

    const existingIndex =
      db.listings.findIndex(
        x => String(x.id) === String(listing.id)
      );

    if (existingIndex >= 0) {
      db.listings[existingIndex] =
        listing;
    } else {
      db.listings.push(listing);
    }
  }

  db.pendingPayments =
    db.pendingPayments.filter(
      x => String(x.id) !== String(paymentId)
    );

  writeDb_(db);

  return {
    ok: true,
    message: "Payment confirmed. Listing is now LIVE."
  };
}


function cancelPayment_(paymentId) {

  const db = readDb_();

  const payment =
    db.pendingPayments.find(
      x => String(x.id) === String(paymentId)
    );

  if (!payment) {
    return {
      ok: false,
      error: "Payment notification not found."
    };
  }

  payment.status = "cancelled";

  payment.cancelledAt =
    new Date().toISOString();

  db.pendingPayments =
    db.pendingPayments.filter(
      x => String(x.id) !== String(paymentId)
    );

  writeDb_(db);

  return {
    ok: true,
    message: "Payment cancelled and pending listing destroyed."
  };
}


/* =========================
   LISTINGS
========================= */

function updateListing_(listing) {

  if (!listing || !listing.id) {
    return {
      ok: false,
      error: "Listing ID is required."
    };
  }

  const db = readDb_();

  const index =
    db.listings.findIndex(
      x => String(x.id) === String(listing.id)
    );

  if (index < 0) {
    return {
      ok: false,
      error: "Listing not found."
    };
  }

  const old = db.listings[index];

  db.listings[index] = {
    ...old,
    ...listing,
    id: old.id,
    status: "live",
    updatedAt:
      new Date().toISOString()
  };

  writeDb_(db);

  return {
    ok: true,
    listing: db.listings[index]
  };
}


function deleteListing_(listingId) {

  const db = readDb_();

  const before =
    db.listings.length;

  db.listings =
    db.listings.filter(
      x => String(x.id) !== String(listingId)
    );

  if (db.listings.length === before) {
    return {
      ok: false,
      error: "Listing not found."
    };
  }

  writeDb_(db);

  return {
    ok: true,
    message: "Listing deleted."
  };
}


/* =========================
   REQUIREMENTS
========================= */

function updateRequirement_(req) {

  if (!req || !req.id) {
    return {
      ok: false,
      error: "Requirement ID is required."
    };
  }

  const db = readDb_();

  const index =
    db.reqs.findIndex(
      x => String(x.id) === String(req.id)
    );

  if (index < 0) {
    return {
      ok: false,
      error: "Requirement not found."
    };
  }

  db.reqs[index] = {
    ...db.reqs[index],
    ...req,
    id: db.reqs[index].id,
    updatedAt:
      new Date().toISOString()
  };

  writeDb_(db);

  return {
    ok: true,
    req: db.reqs[index]
  };
}


function deleteRequirement_(reqId) {

  const db = readDb_();

  const before =
    db.reqs.length;

  db.reqs =
    db.reqs.filter(
      x => String(x.id) !== String(reqId)
    );

  if (db.reqs.length === before) {
    return {
      ok: false,
      error: "Requirement not found."
    };
  }

  writeDb_(db);

  return {
    ok: true,
    message: "Requirement deleted."
  };
}


/* =========================
   ADMIN TOKEN
========================= */

function makeToken_() {

  const timestamp =
    String(Date.now());

  const signature =
    sign_(timestamp);

  return timestamp + "." + signature;
}


function sign_(value) {

  const raw =
    Utilities.computeDigest(
      Utilities.DigestAlgorithm.SHA_256,
      value + "|" + ADMIN_NAME + "|" + ADMIN_PASSWORD,
      Utilities.Charset.UTF_8
    );

  return raw
    .map(function(byte) {

      const v =
        byte < 0
          ? byte + 256
          : byte;

      return ("0" + v.toString(16))
        .slice(-2);

    })
    .join("");
}


function verifyToken_(token) {

  try {

    if (!token) {
      return false;
    }

    const parts =
      String(token).split(".");

    if (parts.length !== 2) {
      return false;
    }

    const ts =
      Number(parts[0]);

    if (!ts) {
      return false;
    }

    if (
      Date.now() - ts >
      TOKEN_TTL_MS
    ) {
      return false;
    }

    if (Date.now() < ts) {
      return false;
    }

    return (
      parts[1] ===
      sign_(parts[0])
    );

  } catch (err) {
    return false;
  }
}


/* =========================
   JSON RESPONSE
========================= */

function json_(body) {

  return ContentService
    .createTextOutput(
      JSON.stringify(body)
    )
    .setMimeType(
      ContentService.MimeType.JSON
    );
}
