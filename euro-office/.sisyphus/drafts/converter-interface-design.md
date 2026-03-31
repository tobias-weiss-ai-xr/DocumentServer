# Drawing Converter Interface Design

## Overview

This document describes the refactoring plan for `ASCOfficeDrawingConverter.cpp` (6,026 lines) to break it into modular, maintainable components.

## Current State Analysis

### File: `core/OOXML/PPTXFormat/DrawingConverter/ASCOfficeDrawingConverter.cpp`
- **Size**: 6,026 lines
- **Class**: `CDrawingConverter`
- **Responsibilities Identified**:

1. **Shape Conversion** (lines 1816-2536)
   - `ConvertShape()` - Main shape conversion logic
   - `ConvertWordArtShape()` - Word art special handling
   - `ConvertGroup()` - Group shape conversion

2. **Drawing Element Conversion** (lines 1757-1815)
   - `ConvertDrawing()` - Top-level drawing container
   - `ConvertVml()` - VML format conversion

3. **Visual Markup Language (VML) Conversion** (lines 5532-5696)
   - `ConvertPicVML()` - Picture VML
   - `ConvertShapeVML()` - Shape VML
   - `ConvertGroupVML()` - Group VML
   - `ConvertParaVML()` - Paragraph VML
   - `ConvertTextVML()` - Text VML
   - `ConvertMainPropsToVML()` - Main properties to VML

4. **Property Handling** (lines 1146-1255)
   - `CElementProps` class - Property management
   - `CopyProperty()` - Property copying logic
   - Main properties extraction and management

5. **Color & Styling** (lines 4294-4846)
   - `ConvertColor()` - Color conversion
   - `CheckBrushShape()` - Brush/fill handling
   - `CheckPenShape()` - Pen/stroke handling
   - `CheckBorderShape()` - Border handling
   - `CheckEffectShape()` - Effects handling

6. **Coordinate & Size Management** (lines 3417-3500)
   - `LoadCoordPos()` - Position loading
   - `LoadCoordSize()` - Size loading

7. **Object Serialization** (lines 5165-5531)
   - `SaveObject()` - Object saving
   - `SaveObjectEx()` - Extended object saving
   - `SaveObjectBackground()` - Background saving
   - `SaveObjectExWriterInit/Release()` - Writer management

8. **File & Path Management** (lines 5962-6019)
   - Path setters (Src, Dst, Temp, Embed, Media)
   - Font management
   - Rels management
   - Content relations

9. **Input Processing** (lines 1304-1395)
   - `AddBinData()` - Binary data addition
   - `AddShapeType()` - Shape type addition
   - `AddObject()` - Object addition
   - `ParceObject()` - Object parsing

## Refactoring Strategy

### Target Architecture: Plugin-Based Modular Design

```
DrawingConverter/
├── IDrawingConverter.h           # Abstract interface
├── DrawingConverterFactory.h     # Factory for creating converters
├── Converters/
│   ├── IShapeConverter.h         # Shape conversion interface
│   ├── ShapeConverter.cpp        # Main shape conversion logic
│   ├── GroupConverter.h          # Group shape handling
│   ├── WordArtConverter.h        # Word art special handling
│   └── PictureConverter.h        # Picture/image handling
├── VML/
│   ├── IVMLConverter.h           # VML conversion interface
│   ├── VMLConverter.cpp          # VML implementation
│   ├── VMLShapeConverter.h       # Shape VML
│   ├── VMLPictureConverter.h     # Picture VML
│   ├── VMLGroupConverter.h       # Group VML
│   └── VMLTextConverter.h        # Text/paragraph VML
├── Styling/
│   ├── IStyleConverter.h         # Style conversion interface
│   ├── ColorConverter.cpp        # Color conversion
│   ├── BrushConverter.cpp        # Brush/fill conversion
│   ├── PenConverter.cpp          # Pen/stroke conversion
│   ├── BorderConverter.cpp       # Border conversion
│   └── EffectConverter.cpp       # Effects conversion
├── Properties/
│   ├── IPropertyManager.h        # Property management interface
│   ├── PropertyManager.cpp       # Property handling
│   └── ElementProps.cpp          # Element properties
├── Geometry/
│   ├── IGeometryHandler.h        # Geometry interface
│   ├── GeometryHandler.cpp       # Position/size handling
│   └── CoordinateSystem.h        # Coordinate utilities
├── Serialization/
│   ├── ISerializer.h             # Serialization interface
│   ├── ObjectSerializer.cpp      # Object saving logic
│   └── WriterManager.cpp         # XML writer management
└── Utils/
    ├── PathManager.h             # Path/file management
    └── ConversionUtils.h         # Utility functions
```

### Interface Design

#### 1. IDrawingConverter (Abstract Base)

```cpp
class IDrawingConverter
{
public:
    virtual ~IDrawingConverter() = default;
    
    // Core conversion entry point
    virtual void Convert(const ConversionContext& context) = 0;
    
    // Configuration
    virtual void SetSourcePath(const std::wstring& path) = 0;
    virtual void SetDestinationPath(const std::wstring& path) = 0;
    virtual void SetTempPath(const std::wstring& path) = 0;
    
    // Lifecycle
    virtual void Clear() = 0;
    
    // Dependencies injection
    virtual void SetFontManager(NSFonts::IFontManager* manager) = 0;
    virtual void SetFontPicker(COfficeFontPicker* picker) = 0;
};
```

#### 2. IShapeConverter

```cpp
class IShapeConverter
{
public:
    virtual ~IShapeConverter() = default;
    
    virtual void ConvertShape(PPTX::Logic::SpTreeElem* elem,
                             const XmlUtils::CXmlNode& node,
                             ConversionContext& context) = 0;
    
    virtual void ConvertGroup(PPTX::Logic::SpTreeElem* result,
                             const XmlUtils::CXmlNode& node,
                             ConversionContext& context) = 0;
    
    virtual void ConvertWordArt(PPTX::Logic::SpTreeElem* elem,
                               const XmlUtils::CXmlNode& node,
                               CPPTShape* shape) = 0;
};
```

#### 3. IVMLConverter

```cpp
class IVMLConverter
{
public:
    virtual ~IVMLConverter() = default;
    
    virtual void ConvertToVml(const ConversionContext& context,
                             NSBinPptxRW::CXmlWriter& writer) = 0;
    
    virtual void ConvertShapeVml(PPTX::Logic::SpTreeElem& elem,
                                const std::wstring& mainProps,
                                NSBinPptxRW::CXmlWriter& writer) = 0;
    
    virtual void ConvertPictureVml(PPTX::Logic::SpTreeElem& elem,
                                  const std::wstring& mainProps,
                                  NSBinPptxRW::CXmlWriter& writer) = 0;
    
    virtual void ConvertGroupVml(PPTX::Logic::SpTreeElem& elem,
                                const std::wstring& mainProps,
                                NSBinPptxRW::CXmlWriter& writer) = 0;
    
    virtual void ConvertTextVml(XmlUtils::CXmlNode& textBox,
                               PPTX::Logic::Shape* shape) = 0;
};
```

#### 4. IStyleConverter

```cpp
class IStyleConverter
{
public:
    virtual ~IStyleConverter() = default;
    
    virtual void ConvertColor(const PPTX::Logic::UniColor& color,
                             nullable_string& sColor,
                             nullable_string& sOpacity) = 0;
    
    virtual void ConvertBrush(PPTX::Logic::SpTreeElem* elem,
                             const XmlUtils::CXmlNode& node,
                             CPPTShape* shape) = 0;
    
    virtual void ConvertPen(PPTX::Logic::SpTreeElem* elem,
                           const XmlUtils::CXmlNode& node,
                           CPPTShape* shape) = 0;
    
    virtual void ConvertBorder(PPTX::Logic::SpTreeElem* elem,
                              const XmlUtils::CXmlNode& node,
                              CPPTShape* shape) = 0;
    
    virtual void ConvertEffects(PPTX::Logic::SpTreeElem* elem,
                               const XmlUtils::CXmlNode& node,
                               CPPTShape* shape) = 0;
};
```

### Module Responsibilities

| Module | Responsibility | Lines (Current) | Target Size |
|--------|---------------|-----------------|-------------|
| ShapeConverter | Shape conversion logic | ~2500 | <800 |
| VMLConverter | VML format conversion | ~1200 | <500 |
| StyleConverter | Color, brush, pen, border | ~1500 | <600 |
| PropertyManager | Property handling | ~400 | <300 |
| GeometryHandler | Position/size | ~150 | <200 |
| ObjectSerializer | Serialization | ~700 | <400 |
| PathManager | File/path management | ~100 | <150 |
| **Total** | | **6026** | **~3000** (distributed) |

### Migration Strategy

#### Phase 1: Extract Interfaces (Task 15)
- Define all interfaces (IDrawingConverter, IShapeConverter, etc.)
- Create factory pattern for converter creation
- Document module boundaries

#### Phase 2: Extract Logic (Task 16)
- Move shape conversion to ShapeConverter
- Move VML logic to VMLConverter
- Move styling to StyleConverter
- Keep original class as delegator

#### Phase 3: Split God Class (Task 17)
- Remove logic from CDrawingConverter
- Replace with delegations to new modules
- Reduce original file to <500 lines

#### Phase 4: Plugin Architecture (Task 18)
- Implement plugin loading mechanism
- Create plugin interface (IDrawingConverterPlugin)
- Support dynamic format handlers

#### Phase 5: Build System (Task 19)
- Update CMakeLists.txt/Makefile
- Create separate build targets
- Verify incremental builds

## Backward Compatibility

### API Preservation
- All public methods of `CDrawingConverter` must remain unchanged
- New modules internal implementation details
- Use delegation pattern during transition

### Migration Path
1. Phase 1-2: Coexistence (old + new)
2. Phase 3: Gradual migration
3. Phase 4: Full plugin support
4. Phase 5: Deprecate old API (optional)

## Testing Strategy

### Unit Tests
- Test each converter module independently
- Mock dependencies using interfaces
- Cover edge cases (empty shapes, invalid XML)

### Integration Tests
- End-to-end conversion tests
- Compare output with baseline
- Test with real-world documents

### Regression Tests
- Maintain existing test suite
- Ensure no behavior changes
- Automated comparison of outputs

## Risk Mitigation

### Identified Risks
1. **Breaking existing functionality**
   - Mitigation: Extensive testing, gradual migration
   - Rollback: Keep original implementation until verified

2. **Performance degradation**
   - Mitigation: Benchmark before/after
   - Optimize hot paths if needed

3. **Circular dependencies**
   - Mitigation: Clear module boundaries, dependency injection
   - Use interfaces to decouple

4. **Loss of context/state**
   - Mitigation: ConversionContext object carries all state
   - Explicit parameter passing

## Success Criteria

- [ ] Original file reduced from 6,026 to <500 lines
- [ ] 6-8 focused modules created (<800 lines each)
- [ ] All public APIs preserved (backward compatible)
- [ ] No increase in build time
- [ ] All existing tests pass
- [ ] Code coverage maintained or improved
- [ ] Static analysis shows no new warnings
- [ ] Plugin architecture supports dynamic loading

## Timeline Estimate

- Task 15 (Design): 1-2 days
- Task 16 (Extract): 3-4 days
- Task 17 (Split): 2-3 days
- Task 18 (Plugin): 2-3 days
- Task 19 (Build): 1 day
- **Total**: 9-13 days

## References

- Original file: `core/OOXML/PPTXFormat/DrawingConverter/ASCOfficeDrawingConverter.cpp`
- Header: `core/OOXML/PPTXFormat/DrawingConverter/ASCOfficeDrawingConverter.h`
- Related: Tasks 5-9 (security fixes must be preserved)

---

**Document Version**: 1.0
**Created**: 2026-03-31
**Author**: Prometheus Planning Agent
**Status**: Design Complete - Ready for Implementation
