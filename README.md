# swift-app


//
//  ContentView.swift
//  Project
//
//  Created by Talin Harun on 06.01.2025.
//


import SwiftUI
import MapKit
import CoreLocation

extension CLLocationCoordinate2D: @retroactive Equatable {
    public static func == (lhs: CLLocationCoordinate2D, rhs: CLLocationCoordinate2D) -> Bool {
        return lhs.latitude == rhs.latitude && lhs.longitude == rhs.longitude
    }
}

extension Color {
    static let lightBrown = Color(red: 0.87, green: 0.7, blue: 0.5) // Light Brown
    static let burgundy = Color(red: 0.5, green: 0.0, blue: 0.13) // Custom Burgundy Color
}


struct InitialView: View {
    @State private var preferences: [String: Bool] = [
        "Atatürk": false, "Train": false, "Plane": false, "Car": false, "Toys": false,
        "Period": false, "Science": false, "Ship": false, "Comms": false, "Motor": false,
        "Ferry": false
    ]

    var body: some View {
        NavigationView {
            VStack(spacing: 10) {
                Spacer()

                Text("Rahmi Koç Museum Navigation App")
                    .font(.system(size: 45, weight: .bold))
                    .foregroundColor(Color.burgundy)
                    .multilineTextAlignment(.center)

                Spacer()

                VStack(spacing: 15) {
                    NavigationLink(destination: ContentView()) {
                        Text("Go to Map View")
                            .frame(maxWidth: .infinity)
                            .padding()
                            .background(Color.burgundy)
                            .foregroundColor(.white)
                            .cornerRadius(12)
                    }

                    NavigationLink(destination: PreferencesView(preferences: $preferences)) {
                        Text("Set Preferences")
                            .frame(maxWidth: .infinity)
                            .padding()
                            .background(Color.burgundy)
                            .foregroundColor(.white)
                            .cornerRadius(12)
                    }

                    NavigationLink(destination: ContentPage()) {
                        Text("Go to Content Page")
                            .frame(maxWidth: .infinity)
                            .padding()
                            .background(Color.burgundy)
                            .foregroundColor(.white)
                            .cornerRadius(12)
                    }
                }
                .padding(.horizontal, 30)

                Spacer()
            }
            .background(Color.white.edgesIgnoringSafeArea(.all))
        }
    }
}
struct InitialView_Previews: PreviewProvider {
    static var previews: some View {
        NavigationView {
            InitialView()
        }
    }
}


struct MapView: UIViewRepresentable {
    let annotations: [MuseumSectionAnnotation]
    @Binding var zoomLevel: MKCoordinateSpan

    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        let coordinator = context.coordinator
        coordinator.mapView = mapView // Pass mapView reference to Coordinator
        mapView.delegate = coordinator

        // Center the map on Rahmi Koç Museum
        let museumCenter = CLLocationCoordinate2D(latitude: 41.04241, longitude: 28.94881)
        let region = MKCoordinateRegion(center: museumCenter, span: zoomLevel)
        mapView.setRegion(region, animated: false)

        // Add annotations
        mapView.addAnnotations(annotations)

        return mapView
    }

    func updateUIView(_ uiView: MKMapView, context: Context) {
        uiView.removeAnnotations(uiView.annotations)
        uiView.addAnnotations(annotations)

        let museumCenter = CLLocationCoordinate2D(latitude: 41.0421, longitude: 28.9497)
        let updatedRegion = MKCoordinateRegion(center: museumCenter, span: zoomLevel)
        uiView.setRegion(updatedRegion, animated: true)
    }

    func makeCoordinator() -> Coordinator {
        Coordinator()
    }

    class Coordinator: NSObject, MKMapViewDelegate {
        var selectedAnnotations = [MuseumSectionAnnotation]()
        var mapView: MKMapView?

        // Compute distances between two coordinates
        private func distance(from: CLLocationCoordinate2D, to: CLLocationCoordinate2D) -> Double {
            let loc1 = CLLocation(latitude: from.latitude, longitude: from.longitude)
            let loc2 = CLLocation(latitude: to.latitude, longitude: to.longitude)
            return loc1.distance(from: loc2)
        }

        // Find the shortest path using a greedy algorithm
        private func shortestPath() -> [CLLocationCoordinate2D] {
            guard selectedAnnotations.count > 1 else { return selectedAnnotations.map { $0.coordinate } }

            var unvisited = selectedAnnotations.map { $0.coordinate }
            var path: [CLLocationCoordinate2D] = []
            var current = unvisited.removeFirst() // Start at the first pin
            path.append(current)

            while !unvisited.isEmpty {
                // Find the nearest unvisited pin
                if let nearest = unvisited.min(by: { distance(from: current, to: $0) < distance(from: current, to: $1) }) {
                    path.append(nearest)
                    current = nearest
                    unvisited.removeAll { $0 == nearest }
                }
            }

            return path
        }

        // Update route with shortest path
        func updateRoute() {
            guard let mapView = mapView else { return }

            // Remove existing overlays
            mapView.removeOverlays(mapView.overlays)

            // Get the shortest path and create a polyline
            let path = shortestPath()
            guard path.count > 1 else { return } // Need at least two points for a route
            let polyline = MKPolyline(coordinates: path, count: path.count)
            mapView.addOverlay(polyline)
        }

        func mapView(_ mapView: MKMapView, rendererFor overlay: MKOverlay) -> MKOverlayRenderer {
            if let polyline = overlay as? MKPolyline {
                let renderer = MKPolylineRenderer(polyline: polyline)
                renderer.strokeColor = .blue
                renderer.lineWidth = 4
                return renderer
            }
            return MKOverlayRenderer()
        }
        
        func mapView(_ mapView: MKMapView, viewFor annotation: MKAnnotation) -> MKAnnotationView? {
            guard let museumAnnotation = annotation as? MuseumSectionAnnotation else { return nil }

            let identifier = "MuseumPin"
            var annotationView = mapView.dequeueReusableAnnotationView(withIdentifier: identifier) as? MKMarkerAnnotationView

            if annotationView == nil {
                annotationView = MKMarkerAnnotationView(annotation: annotation, reuseIdentifier: identifier)
                annotationView?.canShowCallout = true
            } else {
                annotationView?.annotation = annotation
            }

            // Customize appearance based on crowdedness
            switch museumAnnotation.crowdednessLevel {
            case "High":
                annotationView?.markerTintColor = .red
            case "Medium":
                annotationView?.markerTintColor = .orange
            default:
                annotationView?.markerTintColor = .green
            }

            // Highlight selected annotations
            if museumAnnotation.isSelected {
                annotationView?.markerTintColor = .blue
                annotationView?.glyphText = "★"
            } else {
                annotationView?.glyphText = nil
            }

            return annotationView
        }

        func mapView(_ mapView: MKMapView, didSelect view: MKAnnotationView) {
            guard let annotation = view.annotation as? MuseumSectionAnnotation else { return }

            if annotation.crowdednessLevel == "High" {
                // Show warning alert
                showCrowdednessWarning(for: annotation, mapView: mapView)
            } else {
                toggleSelection(for: annotation, in: mapView)
            }
        }

        private func showCrowdednessWarning(for annotation: MuseumSectionAnnotation, mapView: MKMapView) {
            let alert = UIAlertController(
                title: "Warning",
                message: "\(annotation.title ?? "This location") is crowded now. Do you still want to proceed?",
                preferredStyle: .alert
            )
            alert.addAction(UIAlertAction(title: "Yes", style: .default, handler: { _ in
                self.toggleSelection(for: annotation, in: mapView)
            }))
            alert.addAction(UIAlertAction(title: "No", style: .cancel))
            
            // Present the alert
            if let viewController = mapView.window?.rootViewController {
                viewController.present(alert, animated: true)
            }
        }

        private func toggleSelection(for annotation: MuseumSectionAnnotation, in mapView: MKMapView) {
            annotation.isSelected.toggle()

            if annotation.isSelected {
                selectedAnnotations.append(annotation)
            } else {
                selectedAnnotations.removeAll { $0 === annotation }
            }

            updateRoute()

            // Refresh annotation appearance
            mapView.removeAnnotation(annotation)
            mapView.addAnnotation(annotation)
        }

    }
}

class MuseumSectionAnnotation: NSObject, MKAnnotation {
    let coordinate: CLLocationCoordinate2D
    let title: String?
    let subtitle: String?
    var isSelected: Bool = false
    var crowdednessLevel: String

    init(coordinate: CLLocationCoordinate2D, title: String?, subtitle: String?) {
        self.coordinate = coordinate
        self.title = title
        self.subtitle = subtitle
        self.crowdednessLevel = ["Low", "Medium", "High"].randomElement()! // Assign random level
    }
}




struct ContentView: View {
    @State private var zoomLevel = MKCoordinateSpan(latitudeDelta: 0.002, longitudeDelta: 0.002)
    @State private var currentFloor: Floor = .groundLevel
    @State private var showLegend = false // State to toggle the legend view

    enum Floor {
        case groundLevel, firstFloor, basementLevel
    }

    // Annotations for different levels
    let groundLevelAnnotations = [
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04189, longitude: 28.94845), title: "DC-3 Yolcu Uçağı", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04193, longitude: 28.94849), title: "Müze Mağaza", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04204, longitude: 28.94814), title: "Berlin 65 Vagonu", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04241, longitude: 28.94882), title: "Turgut Alp Vinci", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04230, longitude: 28.94863), title: "Buhar Makinası", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04224, longitude: 28.94855), title: "Elmalı Barajı Pompası", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04250, longitude: 28.94832), title: "Açık Teşhir Alanı", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04267, longitude: 28.94847), title: "B-24 Liberator", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04251, longitude: 28.94804), title: "Seka Vinci", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04238, longitude: 28.94925), title: "F-104 Savaş Uçağı", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04127, longitude: 28.94840), title: "Halat Restaurant", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04132, longitude: 28.94820), title: "Fenerbahçe Vapuru", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04195, longitude: 28.94906), title: "Aydın Çubukçu Galerisi", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04196, longitude: 28.94939), title: "Amral Teknesi", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04192, longitude: 28.94932), title: "Sayanora Filikası", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04221, longitude: 28.94951), title: "T02", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04208, longitude: 28.94973), title: "T03", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04198, longitude: 28.94960), title: "T06", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04148, longitude: 28.94958), title: "T11", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04136, longitude: 28.94943), title: "T12", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04128, longitude: 28.94931), title: "T13", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04119, longitude: 28.94921), title: "T14", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04124, longitude: 28.94899), title: "T15", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04180, longitude: 28.94950), title: "T17", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04164, longitude: 28.94920), title: "T17a", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04167, longitude: 28.94924), title: "T17b", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04170, longitude: 28.94928), title: "T17c", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04173, longitude: 28.94932), title: "T17d", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04176, longitude: 28.94936), title: "T17e", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04179, longitude: 28.94940), title: "T17f", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04182, longitude: 28.94944), title: "T17g", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04186, longitude: 28.94950), title: "T18", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04162, longitude: 28.94875), title: "T19a", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04167, longitude: 28.94880), title: "T19b", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04172, longitude: 28.94885), title: "T19c", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04177, longitude: 28.94890), title: "T19d", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04214, longitude: 28.94899), title: "T20", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04250, longitude: 28.94972), title: "Keşif Küresi", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04282, longitude: 28.94924), title: "Suzy’s Cade Du Levant", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04299, longitude: 28.94911), title: "Sergi Salonu", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04268, longitude: 28.94949), title: "L01", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04273, longitude: 28.94954), title: "L02", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04278, longitude: 28.94959), title: "L03", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04259, longitude: 28.94962), title: "L04", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04264, longitude: 28.94967), title: "L05", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04269, longitude: 28.94972), title: "L06", subtitle: nil),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04268, longitude: 28.94961), title: "L07", subtitle: nil),
    ]

    let firstFloorAnnotations = [
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04213, longitude: 28.94934), title: "T01", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04207, longitude: 28.94971), title: "T04", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04221, longitude: 28.94966), title: "T05", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04177, longitude: 28.94995), title: "T07", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04170, longitude: 28.94986), title: "T08", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04148, longitude: 28.94958), title: "T10", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04119, longitude: 28.94921), title: "T14", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04199, longitude: 28.94918), title: "T19e", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04176, longitude: 28.94892), title: "T19e", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04269, longitude: 28.94972), title: "L08", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04264, longitude: 28.94967), title: "L09", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04259, longitude: 28.94962), title: "L10", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04263, longitude: 28.94956), title: "L11", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04268, longitude: 28.94949), title: "L12", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04273, longitude: 28.94954), title: "L13", subtitle: "Upstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04278, longitude: 28.94959), title: "L14", subtitle: "Upstairs"),
    ]

    let basementLevelAnnotations = [
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04297, longitude: 28.94908), title: "T15", subtitle: "Downstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04292, longitude: 28.94914), title: "T16", subtitle: "Downstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04300, longitude: 28.94915), title: "T17", subtitle: "Downstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04291, longitude: 28.94929), title: "T18", subtitle: "Downstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04284, longitude: 28.94941), title: "T19", subtitle: "Downstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04281, longitude: 28.94927), title: "T19", subtitle: "Downstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04272, longitude: 28.94949), title: "T20", subtitle: "Downstairs"),
        MuseumSectionAnnotation(coordinate: CLLocationCoordinate2D(latitude: 41.04264, longitude: 28.94964), title: "T21", subtitle: "Downstairs"),
    ]

    var currentAnnotations: [MuseumSectionAnnotation] {
        switch currentFloor {
        case .groundLevel:
            return groundLevelAnnotations
        case .firstFloor:
            return firstFloorAnnotations
        case .basementLevel:
            return basementLevelAnnotations
        }
    }
    struct LegendView: View {
        @Environment(\.dismiss) var dismiss // Access the dismiss environment action
        
        var body: some View {
            NavigationView {
                List {
                    NavigationLink("MUSTAFA V. KOÇ BİNASI (Mustafa V. Koç Building)", destination: MustafaKocView())
                    NavigationLink("TERSHANE (Shipyard)", destination: TershaneView())
                }
                .navigationTitle("Legend")
                .toolbar {
                    ToolbarItem(placement: .navigationBarTrailing) {
                        Button("Close") {
                            dismiss() // Dismiss the sheet
                        }
                    }
                }
                .foregroundColor(.burgundy) // Set title color to burgundy
            }
        }
    }

    struct MustafaKocView: View {
        var body: some View {
            NavigationView {
                List {
                    NavigationLink("ALT KAT (Downstairs)", destination: MustafaKocAlt())
                    NavigationLink("GİRİŞ KATI (Entrance Floor)", destination: MustafaKocGiris())
                    NavigationLink("ÜST KAT (Upper Floor)", destination: MustafaKocUst())
                }
                .navigationTitle("MUSTAFA V. KOÇ")
            }
            .foregroundColor(.burgundy) // Set title color to burgundy

        }
    }

    struct TershaneView: View {
        var body: some View {
            NavigationView {
                List {
                    NavigationLink("GİRİŞ KATI (Entrance Floor)", destination: TershaneGiris())
                    NavigationLink("ÜST KAT (Upper Floor)", destination: TershaneUst())
                    NavigationLink("AÇIK TEŞHİR ALANI (Open-Air Exhibition Area)", destination: TershaneAcik())
                }
                .navigationTitle("TERSHANE")
            }
            .foregroundColor(.burgundy) // Set title color to burgundy

        }
    }

    struct MustafaKocAlt: View {
        var body: some View {
            List {
                Text("L15 : Havacılık (Aviation)")
                Text("L16 : Restorasyon Atölyesi (Restoration Workshop)")
                Text("L17 : Lokomotif ve Otomobil Modelleri (Locomotive and Car Models)")
                Text("L18 : Oyuncaklar (Toys)")
                Text("L19 : Denizcilik Modelleri (Maritime Models)")
                Text("L20 : Sinema Bölümü (Cinema Section)")
                Text("L21 : Matbaa Makineleri (Printing Machines)")
            }
            .navigationTitle("ALT KAT (Downstairs)")
        }
    }

    struct MustafaKocGiris: View {
        var body: some View {
            List {
                Text("L01 : Buharlı Makine Modelleri (Steam Engine Models)")
                Text("L02 : Buharlı Gemi Makine Modelleri (Steamship Engine Models)")
                Text("L03 : Sıcak Hava ve İçten Yanmalı Motor Modelleri (Hot Air and Internal Combustion Engine Models)")
                Text("L04 : Buharlı Makine Modelleri (Steam Engine Models)")
                Text("L05 : Buharlı Silindirler ve Çekici Makine Modelleri (Steam Cylinders and Tractor Engine Models)")
                Text("L06 : Buharlı Makine Modelleri (Steam Engine Models)")
                Text("L07 : Lokomotif Modelleri ve Kalender Vapuru Makinesi (Locomotive Models and Kalender Steamboat Engine)")
            }
            .navigationTitle("GİRİŞ KATI (Entrance Floor)")
        }
    }

    struct MustafaKocUst: View {
        var body: some View {
            List {
                Text("L08 - L11 : Bilimsel Aletler (Scientific Instruments)")
                Text("L12 - L14 : İletişim Aletleri (Communication Instruments)")
            }
            .navigationTitle("ÜST KAT (Upper Floor)")
        }
    }

    struct TershaneGiris: View {
        var body: some View {
            List {
                Text("T02 : Sualtı Bölümü (Underwater Section)")
                Text("T03 : Astronomi / Enerji / Fen Atölyeleri (Astronomy / Energy / Science Workshops)")
                Text("T06 : Erdoğan Gönül Galerisi | Otomobiller (Erdoğan Gönül Gallery | Automobiles)")
                Text("T11 : Dr. Bülent Bulgurlu Galerisi | Otomobiller (Dr. Bülent Bulgurlu Gallery | Automobiles)")
                Text("T12 : Buhar Makineleri | Dizel Motorları (Steam Engines | Diesel Engines)")
                Text("T13 : Araser Zeytinyağı Fabrikası (Araser Olive Oil Factory)")
                Text("T14 : Marangozhane (Carpentry Shop)")
                Text("T15 : Gemi Makineleri (Ship Engines)")
                Text("T16 : Tarihi Kızak (Historical Sledge)")
                Text("T17 : Nostaljik Dükkanlar (Nostalgic Shops)")
                Text("T17a : Haliç Oyuncakçısı (Golden Horn Toymaker)")
                Text("T17b : Gemi Donatımı (Ship Fitting)")
                Text("T17c : Dakik Saat (Clockmaker)")
                Text("T17d : Dövme Demir (Blacksmith)")
                Text("T17e : Ismarlama Kundura (Custom Shoes)")
                Text("T17f : Fecri Aletler (Scientific Tools)")
                Text("T17g : Şifa Eczanesi (Healing Pharmacy)")
                Text("T18 : Gemi Buhar Makinesi (Ship Steam Engine)")
                Text("T19 : Denizcilik (Maritime)")
                Text("T19a : Balıkçı Barınağı (Fishermen's Harbor)")
                Text("T19b : Tekneler (Boats)")
                Text("T19c : Kosta Usta Motor Tamir Atölyesi (Kosta Usta Motor Repair Workshop)")
                Text("T19d : Ayvansaray Sandal Yapım Atölyesi (Ayvansaray Boat Building Workshop)")
                Text("T20 : Aydın Çubukçu Galerisi - Raylı Ulaşım (Aydın Çubukçu Gallery - Rail Transport)")
            }
            .navigationTitle("GİRİŞ KATI (Entrance Floor)")
        }
    }

    struct TershaneUst: View {
        var body: some View {
            List {
                Text("T01 : Rahmi M. Koç Galerisi Atatürk Koleksiyonu (Rahmi M. Koç Gallery Atatürk Collection)")
                Text("T04 : Renkli Matematik Dünyası (Colorful World of Mathematics)")
                Text("T05 : Anasıfı Eğitim Atölyesi (Preschool Education Workshop)")
                Text("T07 : Motosikletler (Motorcycles)")
                Text("T08 : Bebek Arabaları (Baby Carriages)")
                Text("T09 : Bisikletler (Bicycles)")
                Text("T10 : Kağnılar | At Arabaları | Kızaklar (Ox Carts | Horse Carts | Sledges)")
                Text("T14 : Torna Tezgahları (Lathe Machines)")
                Text("T19e : Kayıklar | Dıştan Takma Motorlar (Boats | Outboard Motors)")
            }
            .navigationTitle("ÜST KAT (Upper Floor)")
        }
    }

    struct TershaneAcik: View {
        var body: some View {
            List {
                Text("Anadol Otomobiller (Anadol Cars)")
                Text("Yarış Otomobilleri (Race Cars)")
                Text("İtfaiye Arabaları (Fire Engines)")
                Text("Traktörler (Tractors)")
                Text("Sovyet Otomobilleri (Soviet Cars)")
                Text("Dört Çekerli Araçlar (Four-Wheel Drive Vehicles)")
                Text("Uçaklar (Aircrafts)")
                Text("B-24 Harley's Harem")
                Text("Jet Provost")
                Text("Hamsa Jet")
                Text("Eğitim Uçağı (Training Aircraft)")
                Text("Tarım Uçağı (Agricultural Aircraft)")
                Text("Atlıkarınca (Carousel)")
                Text("Hasköy Sütlüce Demiryolu İstasyonu (Hasköy Sütlüce Railway Station)")
                Text("Seka Vinci (Seka Crane)")
            }
            .navigationTitle("AÇIK TEŞHİR ALANI (Open-Air Exhibition Area)")
        }
    }

    var body: some View {
        ZStack {
            MapView(annotations: currentAnnotations, zoomLevel: $zoomLevel)
                .edgesIgnoringSafeArea(.all)

            VStack {
                // Floor Switcher Buttons
                HStack {
                    Button(action: { currentFloor = .basementLevel }) {
                        Text("Downstairs")
                            .padding()
                            .background(currentFloor == .basementLevel ? Color.blue : Color.burgundy)
                            .foregroundColor(.white)
                            .clipShape(Capsule())
                    }

                    Button(action: { currentFloor = .groundLevel }) {
                        Text("Ground Level")
                            .padding()
                            .background(currentFloor == .groundLevel ? Color.blue : Color.burgundy)
                            .foregroundColor(.white)
                            .clipShape(Capsule())
                    }

                    Button(action: { currentFloor = .firstFloor }) {
                        Text("Upstairs")
                            .padding()
                            .background(currentFloor == .firstFloor ? Color.blue : Color.burgundy)
                            .foregroundColor(.white)
                            .clipShape(Capsule())
                    }
                }
                .padding()
                .background(Color.burgundy.opacity(0.5))
                .clipShape(Capsule())

                Spacer()

                // Zoom Buttons
                HStack {
                    Button(action: {
                        zoomLevel = MKCoordinateSpan(
                            latitudeDelta: max(zoomLevel.latitudeDelta / 2, 0.0005),
                            longitudeDelta: max(zoomLevel.longitudeDelta / 2, 0.0005)
                        )
                    }) {
                        Image(systemName: "plus.magnifyingglass")
                            .font(.system(size: 30))
                            .foregroundColor(.white)
                            .padding()
                            .background(Color.burgundy.opacity(0.7))
                            .clipShape(Circle())
                    }

                    Button(action: {
                        zoomLevel = MKCoordinateSpan(
                            latitudeDelta: zoomLevel.latitudeDelta * 2,
                            longitudeDelta: zoomLevel.longitudeDelta * 2
                        )
                    }) {
                        Image(systemName: "minus.magnifyingglass")
                            .font(.system(size: 30))
                            .foregroundColor(.white)
                            .padding()
                            .background(Color.burgundy.opacity(0.7))
                            .clipShape(Circle())
                    }
                }
                .padding()
            }
            // Legend Button
                       VStack {
                           Spacer()
                           HStack {
                               Spacer()
                               Button(action: {
                                   showLegend = true
                               }) {
                                   Text("Legend")
                                       .padding()
                                       .background(Color.burgundy.opacity(0.7))
                                       .foregroundColor(.white)
                                       .clipShape(Capsule())
                               }
                               .padding()
                           }
                       }
                   }
                   .sheet(isPresented: $showLegend) {
                       LegendView()
        }
    }
}

// SwiftUI Preview
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}

struct ContentPage: View {
    let exhibits: [Exhibit] = [
        Exhibit(
            name: "Atatürk Section",
            description: "The objects displayed in this section belong to Mustafa Kemal Atatürk (1881-1938), the founder and first President of the Republic of Turkey. The collection was mainly assembled by Colonel Halil Nuri Yurdakul, who played a significant role in the War of Independence and later became part of Atatürk's close circle. The collection was donated by his son Prof. Dr. Yurdakul Yurdakul and his daughter-in-law Mrs. Ayşe Acatay Yurdakul.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7749, longitude: -122.4194)
        ),
        Exhibit(
            name: "Railway Transportation",
            description: "The museum's railway transportation section consists of two parts. It displays railway vehicles, including Sultan Abdülaziz’s Imperial Train and the Kadıköy-Moda Tramway, as well as finely crafted models of locomotives and trams, along with various photos and ephemera related to railways.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7849, longitude: -122.4094)
        ),
        Exhibit(
            name: "Aviation",
            description: "The collection, featuring significant examples of aviation history, includes the Wright Brothers' glider model, the iconic Douglas DC-3, and the F-104S Starfighter fighter jet. This collection can be seen in the Mustafa V. Koç Building and the Open Area.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7949, longitude: -122.3994)
        ),
        Exhibit(
            name: "Road Transportation",
            description: "This section displays rare examples of road transportation evolution from the 1800s to the present, including horse-drawn carriages, coaches, children's carriages, bicycles, motorcycles, agricultural objects, classic cars, car models, fire trucks, and steam-powered road vehicles.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7649, longitude: -122.4294)
        ),
        Exhibit(
            name: "Models and Toys",
            description: "Scale models from the 1700s to the present, many of which are rare examples of their kind, are a significant part of the museum's collection. Prominent model collections related to steam engines, railway transportation, maritime, aviation, and road transportation can be seen in the respective sections. The collection also includes miniature objects and toys from various countries and periods, with toys mostly displayed in the Mustafa V. Koç Building. Various miniature objects and dollhouses, opening a door to a magical world, can be seen in the Shipyard - Main Entrance.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7649, longitude: -122.4294)
        ),
        Exhibit(
            name: "Living Past",
            description: "Sections where 19th-century shops and workshops are recreated in a lifelike manner also display many intriguing pieces from the collection. The sections include a lathe shop, olive oil factory, film set, captain’s cabin, fisherman’s dock, motor repair workshop, boatbuilding workshop, tools shop, pharmacy, shoemaker, blacksmith, clockmaker, ship equipment and toy shops, and more, all strategically located to support the collection.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7649, longitude: -122.4294)
        ),
        Exhibit(
            name: "Scientific Instruments",
            description: "This collection, which includes important observation and measurement instruments such as a 14th-century celestial globe and a 19th-century transit telescope, sheds light on the history of science. The entire collection can be viewed in the Mustafa V. Koç Building.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7649, longitude: -122.4294)
        ),
        Exhibit(
            name: "Maritime",
            description: "The Rahmi M. Koç Museum has an extensive collection of maritime objects and models. In the Shipyard section, there is a collection of models, several full-size boats and yachts, valuable items like outboard motors, and a rare 'Amphicar.' Among the museum’s most admired items is the impressive Bosphorus Touring Boat, along with smaller boats, canoes, and other small vessels.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7649, longitude: -122.4294)
        ),
        Exhibit(
            name: "Communication Tools",
            description: "The collection, featuring rare examples of important communication tools that emerged with the revolution of communication through the fusion of science and industry, includes items such as the telegraph, telephone, dictaphone, gramophone, camera, and television. This collection can be viewed in the Mustafa V. Koç Building.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7649, longitude: -122.4294)
        ),
        Exhibit(
            name: "Machines",
            description: "The collection of steam and diesel engines, produced both in Turkey and abroad, sheds important light on the development of industry. Notable examples include the steam engine of the Kalender Ship in the Mustafa V. Koç Building and the Marshall portable steam engine in the Shipyard Building.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7649, longitude: -122.4294)
        ),
        Exhibit(
            name: "Road Transportation (Additional Collection)",
            description: "This section contains various collections such as the Anamorphosis Rahmi M. Koç Portrait, Mehmet Memduh Önger Marklin Train Collection, Raoul Cabib Collection, and many others.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7649, longitude: -122.4294)
        ),
        Exhibit(
            name: "Fenerbahçe Ferry",
            description: "The Fenerbahçe Ferry, built in 1952 in Glasgow, Scotland, at the William Denny & Brothers Dumbarton yard, was put into service on May 14, 1953, with the Şirket-i Hayriye (now known as the Turkish Maritime Lines). For many years, it operated between Sirkeci, the Islands, Yalova, and Çınarcık. The ferry, equipped with two 1,500-horsepower Sulzer diesel engines, has a large chimney and wooden components. Since its arrival at the Rahmi M. Koç Museum in 2009, it has served as a museum ferry and is home to the Yalvaç Ural Toy Collection, temporary exhibits, and museum educational activities. Visitors can also enjoy the nostalgic café onboard, offering pleasant moments overlooking the Golden Horn. The ferry is periodically granted by the Istanbul Metropolitan Municipality.",
            coordinate: CLLocationCoordinate2D(latitude: 37.7649, longitude: -122.4294)
        ),
        // Add more exhibits as needed
    ]


    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 20) {
                Text("  Exhibits")
                                .font(.largeTitle)
                                .fontWeight(.bold)
                                .foregroundColor(Color.burgundy)
                                .padding(.bottom, 20) // Adds space below the title

                            ForEach(exhibits, id: \.name) { exhibit in
                                VStack(alignment: .leading, spacing: 10) {
                                    Text(exhibit.name)
                                        .font(.title2)
                                        .fontWeight(.bold)
                                        .foregroundColor(.burgundy)
                                    
                                    Text(exhibit.description)
                                        .font(.body)
                                        .foregroundColor(.black)
                                        .lineLimit(nil)
                                }
                                .padding(.horizontal)
                            }
                        }
                        .padding(.top)
                    }
                }
}

struct ContentPage_Previews: PreviewProvider {
    static var previews: some View {
        ContentPage()
    }
}


struct Exhibit {
    let name: String
    let description: String // This comes before coordinate in the initializer
    let coordinate: CLLocationCoordinate2D
}


struct PreferencesView: View {
    @Binding var preferences: [String: Bool]
    @State private var showSuggestedRooms = false // New state to control navigation
    
    var body: some View {
        NavigationView {
            Form {
                Toggle("Would you like to learn more about Atatürk?", isOn: binding(for: "Atatürk"))
                Toggle("Would you like to see the railway transports and trains?", isOn: binding(for: "Train"))
                Toggle("Are you interested in aircrafts?", isOn: binding(for: "Plane"))
                Toggle("Would you like to see the highway transports and heavy vehicles?", isOn: binding(for: "Car"))
                Toggle("Would you like to see the toys collection?", isOn: binding(for: "Toys"))
                Toggle("Would you like to have a periodical experience?", isOn: binding(for: "Period"))
                Toggle("Are you interested in scientific gadgets?", isOn: binding(for: "Science"))
                Toggle("Are you interested in sea transportations and vessels?", isOn: binding(for: "Ship"))
                Toggle("Are you interested in history of communication?", isOn: binding(for: "Comms"))
                Toggle("Would you like to see the motors collection?", isOn: binding(for: "Motor"))
                Toggle("Would you like to see the Fenerbahçe ferry?", isOn: binding(for: "Ferry"))
            }
            .navigationTitle("Preferences")
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("OK") {
                        showSuggestedRooms = true
                    }
                    .foregroundColor(Color.burgundy)
                }
            }
            .sheet(isPresented: $showSuggestedRooms) {
                SuggestedRoomsView(preferences: preferences)
            }
            
        }
    }

    private func binding(for key: String) -> Binding<Bool> {
        Binding(
            get: { preferences[key] ?? false },
            set: { preferences[key] = $0 }
        )
    }
}

struct SuggestedRoomsView: View {
    let preferences: [String: Bool]
    
    // Map preferences to suggested rooms
    private var suggestedRooms: [String] {
        var rooms: [String] = []
        
        if preferences["Atatürk"] == true { rooms.append("T01") }
        if preferences["Train"] == true { rooms.append(contentsOf: ["L17", "L01", "L04", "L06", "L07", "T20"]) }
        if preferences["Plane"] == true { rooms.append(contentsOf: ["L15", "DC-3 yolcu uçağı", "Açık Teşhir alanı", "b-24 Liberatör", "F-104 savaş uçağı"]) }
        if preferences["Car"] == true { rooms.append(contentsOf: ["L17", "L05", "T06", "T11", "T07", "T09", "T10"]) }
        if preferences["Toys"] == true { rooms.append("L18") }
        if preferences["Period"] == true { rooms.append(contentsOf: ["L16", "L20", "T13", "T14", "T17(A-G)"]) }
        if preferences["Science"] == true { rooms.append(contentsOf: ["L08", "L09", "L10", "L11", "T03"]) }
        if preferences["Ship"] == true { rooms.append(contentsOf: ["L19", "L02", "L07", "T02", "T15", "T18", "T19(A-D)"]) }
        if preferences["Comms"] == true { rooms.append(contentsOf: ["L12", "L13", "L14", "L21"]) }
        if preferences["Motor"] == true { rooms.append(contentsOf: ["L03", "T12", "T19E"]) }
        if preferences["Ferry"] == true { rooms.append("Fenerbahçe Vapuru") }
        
        return rooms
    }
    
    var body: some View {
        NavigationView {
            List(suggestedRooms, id: \.self) { room in
                Text(room)
            }
            .navigationTitle("Suggested Rooms")
        }
    }
}


struct MapViewRepresentable: UIViewRepresentable {
    @Binding var mapView: MKMapView
    @Binding var annotations: [MKPointAnnotation]
    @Binding var overlays: [MKOverlay]
    
    func makeUIView(context: Context) -> MKMapView {
        mapView.delegate = context.coordinator
        let tapGesture = UITapGestureRecognizer(target: context.coordinator, action: #selector(Coordinator.handleMapTap(_:)))
        mapView.addGestureRecognizer(tapGesture)
        return mapView
    }
    
    func updateUIView(_ uiView: MKMapView, context: Context) {
        uiView.removeAnnotations(uiView.annotations)
        uiView.addAnnotations(annotations)
        
        uiView.removeOverlays(uiView.overlays)
        uiView.addOverlays(overlays)
    }
    
    func makeCoordinator() -> Coordinator {
        Coordinator(mapView: $mapView, annotations: $annotations, overlays: $overlays)
    }
    class Coordinator: NSObject, MKMapViewDelegate {
        @Binding var mapView: MKMapView
        @Binding var annotations: [MKPointAnnotation]
        @Binding var overlays: [MKOverlay]
        
        init(mapView: Binding<MKMapView>, annotations: Binding<[MKPointAnnotation]>, overlays: Binding<[MKOverlay]>) {
            _mapView = mapView
            _annotations = annotations
            _overlays = overlays
        }
        
        @objc func handleMapTap(_ gesture: UITapGestureRecognizer) {
            let location = gesture.location(in: mapView)
            let coordinate = mapView.convert(location, toCoordinateFrom: mapView)
            
            let annotation = MKPointAnnotation()
            annotation.coordinate = coordinate
            annotations.append(annotation)
            
            if annotations.count >= 2 {
                calculateRoute()
            }
        }
        
        private func calculateRoute() {
            guard annotations.count >= 2 else { return }
            
            var routeRequests: [MKDirections.Request] = []
            
            // Create a request for each segment in the route
            for i in 0..<(annotations.count - 1) {
                let source = annotations[i].coordinate
                let destination = annotations[i + 1].coordinate
                
                let request = MKDirections.Request()
                request.source = MKMapItem(placemark: MKPlacemark(coordinate: source))
                request.destination = MKMapItem(placemark: MKPlacemark(coordinate: destination))
                request.transportType = .walking
                routeRequests.append(request)
            }
            
            // Calculate each route sequentially
            for request in routeRequests {
                let directions = MKDirections(request: request)
                directions.calculate { [weak self] response, error in
                    guard let self = self, let route = response?.routes.first else { return }
                    
                    // Add the route polyline to the map
                    DispatchQueue.main.async {
                        self.mapView.addOverlay(route.polyline)
                    }
                }
            }
        }
        
        func mapView(_ mapView: MKMapView, rendererFor overlay: MKOverlay) -> MKOverlayRenderer {
            if let circleOverlay = overlay as? MKCircle {
                let renderer = MKCircleRenderer(circle: circleOverlay)
                if let level = Int(circleOverlay.title ?? "0") {
                    renderer.fillColor = level == 1 ? UIColor.green.withAlphaComponent(0.4) :
                    level == 2 ? UIColor.yellow.withAlphaComponent(0.4) :
                    UIColor.red.withAlphaComponent(0.4)
                }
                renderer.strokeColor = .black
                renderer.lineWidth = 1
                return renderer
            }
            if let polyline = overlay as? MKPolyline {
                let renderer = MKPolylineRenderer(polyline: polyline)
                renderer.strokeColor = UIColor.blue
                renderer.lineWidth = 4
                return renderer
            }
            return MKOverlayRenderer()
        }
    }
}
