npm install -g newman newman-reporter-htmlextra

# Ejecutar la colección completa con reporte HTML
newman run Booking_Collection.json \
    -e Booking_Environment.json \
    -d Booking_Data.json \
    --reporters htmlextra \
    --reporter-htmlextra-export Reporte_Final.html \
    --reporter-htmlextra-title "Bolivia Booking Go — QA Report"